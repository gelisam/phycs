# phycs

2D physics engine for games, with optional simple rendering.

- [apecs](https://github.com/jonascarpay/apecs) for syntax/interface/memory management with minimal overhead.
- [Chipmunk](https://github.com/slembcke/Chipmunk2D) for C-speed physics.
- [gloss](https://github.com/benl23x5/gloss) for rendering. Can easily be replaced by your own rendering engine.
- [inline-c](https://github.com/fpco/inline-c) for easy binding to Chipmunk.

Run `stack build && stack exec tumbler` for a demonstration.

Still in heavy development.
Feel free to create an issue or PR for suggestions/questions/requests/critiques/spelling fixes/etc.
See [TODO.md](https://github.com/jonascarpay/phycs/blob/master/TODO.md) for suggestions if you want to help out with the code.

### Guided tour
You can run each example with `stack build && stack <example-name>`.

#### helloworld
```haskell
makeWorld "World" [''Color, ''Physics]

initialize = do
  setGlobal $ Gravity (V2 0 (-10))

  newEntity ( DynamicBody
            , Position (V2 0 2)
            , Shape (Circle 0 0.5) defaultProperties {friction = 1} )

  newEntity ( StaticBody
            , Shape (Segment (V2 (-3) 0) (V2 3 0) 0) defaultProperties {friction = 1}
            , Angle (-pi/10) )

main = simulateWorld (InWindow "helloworld" (640,480) (10,10)) 40 initWorld initialize
```
`makeWorld` comes from apecs, `Color` and `InWindow` come from gloss, but are re-exported by `Apecs.Physics` and `Apecs.Physics.Render` respectively.
Adding the `Physics` component to your world gives access to all phycs components.
The first of these is `Gravity`, which is self-explanatory.

When creating a physics object, the first thing you should do is add a `Body`.
A body is either a `DynamicBody`, controlled by forces, a `KinematicBody`, controlled by velocities, or a `StaticBody`, which just has a position and rarely moves.
You can control bodies by setting its lower derivatives, like set a `DynamicBody` position, but not the other way around, and doing so might cause physics artifacts.
For example, setting the velocity of a `DynamicBody` that's touching another object of infinite mass requires the solver to apply an infinite force.

If you don't add a body, setting other properties won't have an effect.
In order to do anything, you probably also want to add a shape.
These can be composed using their `Monoid` instance.

![Screenshot](https://raw.githubusercontent.com/jonascarpay/phycs/master/examples/helloworld.png)


#### tumbler
```haskell
makeWorld "World" [''Color, ''Physics]

initialize = do
  setGlobal $ Gravity (V2 0 (-10))

  newEntity ( KinematicBody
            , AngularVelocity (-pi/6)
            , hollowBox 30 30 0 defaultProperties )

  replicateM_ 400 $ do
    x      <- liftIO$ randomRIO (-9,9)
    y      <- liftIO$ randomRIO (-9,9)
    radius <- liftIO$ randomRIO (0.4,0.8)
    let color = (realToFrac x+9)/19

    newEntity ( DynamicBody
              , Shape (Circle 0 radius) defaultProperties {elasticity=0.9}
              , Position (V2 x y)
              , makeColor 1 color color 1 )

main = simulateWorld (InWindow "phycs" (640,480) (10,10)) 10 initWorld initialize
```
Here we see how a `KinematicBody` preserves its velocity, in this case its angular velocity, even though it contains many `DynamicBody`s.
In a game, this is how you would make things like moving platforms.
A hollowBox is a composition of several line segments.

![Screenshot](https://raw.githubusercontent.com/jonascarpay/phycs/master/examples/tumbler.png)

#### interactive
```haskell
data Box = Box
instance Component Box where
  type Storage Box = Unique Box

makeWorld "World" [''Box, ''Color, ''Physics]

initialize = do
  setGlobal (Gravity (V2 0 (-10)))

  newEntity ( Box
            , KinematicBody
            , hollowBox 30 30 0 defaultProperties )

  replicateM_ 300 $ do
    x      <- liftIO$ randomRIO (-9,9)
    y      <- liftIO$ randomRIO (-9,9)
    radius <- liftIO$ randomRIO (0.4,0.8)
    let color = (realToFrac x+9)/19

    newEntity ( DynamicBody
              , Shape (Circle 0 radius) defaultProperties {elasticity=0.9, friction=1}
              , Position (V2 x y)
              , makeColor 1 color color 1 )

handleEvent (EventKey (SpecialKey KeyLeft)  Down _ _) = rmap $ \ (_ :: Box) -> AngularVelocity (pi/6)
handleEvent (EventKey (SpecialKey KeyRight) Down _ _) = rmap $ \ (_ :: Box) -> AngularVelocity (-pi/6)
handleEvent (EventKey (SpecialKey KeyDown)  Down _ _) = rmap $ \ (_ :: Box) -> AngularVelocity 0
handleEvent _ = return ()

main = playWorld (InWindow "tumbler" (640,480) (10,10)) 10 initWorld handleEvent initialize
```
Interactive version of the tumbler.
We switch to `playWorld` and add an eventHandler.
We could track the entity manually, but it's more idiomatic to tag it with a component.

#### chain
```haskell
makeWorld "World" [''Color, ''Physics]

initialize = do
    setGlobal $ Gravity (V2 0 (-10))
    fixed <- newEntity StaticBody
    chain (cast fixed) 0
  where
    links = 10
    chain a n = do
      b <- newEntity ( DynamicBody
                     , Position (V2 n 5)
                     , Shape (Segment 0 (V2 1 0) 0.1) defaultProperties )

      newEntity $ Constraint (cast a) (cast b) (PivotJoint (V2 n 5))
      unless (n >= links) $ chain b (n+1)

main = simulateWorld (InWindow "chain" (640,480) (10,10)) 30 initWorld initialize
```
A `Constraint` takes as argument two bodies, and a `ConstraintType` between them.
Similar to how adding a `Body` gives you access to `Position`, a Constraint also has a number of properties.
There is nothing preventing you from registering a constraint under the same entity as a body, the underlying `Space` tracks them separately anyway, but it's easiest to make separate entities.

![Screenshot](https://raw.githubusercontent.com/jonascarpay/phycs/master/examples/chain.png)

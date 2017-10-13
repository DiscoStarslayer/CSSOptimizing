# Presentaition notes

## Enabling smooth animations with CSS

### Problems
#### Javascript single threaded
- Compositor + Web Workers run on separate threads

#### Browser Rendering Pipeline
- js -> style -> layout -> paint -> composite
  - js includes anything that results in visual changes
    - Add / Remove dom elements
    - CSS Animations
    - Computing 
    - CSS Transitions
    - Anything that changes how a website looks
  - style
    - Match CSS Rules to selectors
    - Computes all selectors and cascades
  - Layout
    - Use computed rules to determine size of elements and where it is located
    - Has to be rendered proceduraly, nature of css means parent and children elements can affect styles of the element
  - Paint
    - Generate draw calls and rasterize onto surfaces
      - Surfaces known as layers
    - Includes text, colors, images, shadows, ect
  - Compositing
    - Take rasterized layers and compose them onto the screen
    - Needs to be done in correct order to ensure elements are layered ontop of eachother
      - can be overriden with styles like `z-index`
- Generally most visual changes kick off full rendering pipeline
  - Is slow, imagine scrolling a single div across the screen and having to render the entire page for each frame of the animation
- Subset of visual changes only affect the painting step, allowing us to skip layouting
  - Changes that don't change position, size, or the general layout of the page and elements within
  - Changes like modifying the background image or a text color are in this subset
  - Still relatively slow, only skip one step, paint is the slowest step in the pipeline!
- Small subest of visual changes can exist only on the compositing step!
  - Offloads your animation directly to the fast GPU! Skips painting!
  - Separated from the JS thread
  - We want to keep our animations in here
  - Not always easy
  - Keep in CSS transitions if possible
  - Can't avoid JS? Follow FLIP: https://aerotwist.com/blog/flip-your-animations/
  - Still has some problems

#### Compositor Gotchas
- Elements need to be 'promoted'
  - Browsers are lazy by design, if it can avoid it, it will keep your compositor layers as flattened as possible
  - When your component starts moving, it needs to be extracted to a new layer to offload the work to the GPU
  - This process takes computation time as the browser allocates and renders the new layer
- Fix this by pre-promoting elements!
  - Signal to browser this element will move, and to seprate it to it's own layer from the initial page render
    - CSS3 property `will-change`, or a 3D transform on more legacy browsers like `translateZ` will promote the element
  - PROMOTE ALL THE THINGS? `* { will-change: transform }`
  - NO! Compositor layers take up video memory
    - Tight resource, especially on mobile devices
    - Textures needs to be offloaded to GPU, CPU -> GPU pipelines may bottleneck large # of layers
    - Overloading this resource causes thrashing, removing any possible performance gaing
- Limited compositor friendly styles
  - transform
  - opacity

#### Real life example
- DietSUP Drafting Screen
  - Select a player, we do a fun little animation of swapping the heads in the positon cells
    - Headshot overhangs from containing div, tough to create in CSS
    - Smart technique using backgrounds + background positons + border radius
  - Runs great on iOS and desktop, lets ship it :dance_lady:
  - ANDROID >:(
    - Really poor JS and web performance
      - Midrange desktop + 2 year old iPhone easily quadruples top of the line android performance
    - Calculating next positon and rendering all of the athletes for it takes CPU time, adding jank to the animation
  - We gotta fix this
    - Current animation moves a background layer, retriggering painting
    - Re-work animation to exist only on the compositor
    - How to recreate what we have? Get creative!
      - Change border-position animation to a translate3d
      - Build a 'clipping mask' to keep the headshot from clipping out of the bottom but overflowing
      - Create a double buffer with the DOM and each of the players
      - Hide old draft and replace with new one using translate3d
      - Perfect, because we keep this in the GPU thread, our computation doesn't lock up the animation and we get some smooth goodness!
  - Perfect right?
    - SAFARI >:(
      - Apple's browser is super fast and battery efficent, best in the buisness
      - These optimizations can bring about bugs
      - Border radius on our clipping mask isn't working! It shows the athletes shoulders as he moves into the cell
      - How do we fix this?
        - No hard and fast rule, these kind of things you just have to deal with as a web developer
        - Maybe use old animation in safari as safari is fast enough?
          - Garunteed to work but adds extra complexity and older iphones may buckle under the load still
        - Play with css rules!
          - Just fiddle with things, honestly its just a gut thing you get from experience. Eventually found out that if I were to place the clipping mask into the compositor with a translate3d(0,0,0) then safari respects the border-radius!
- Done? Kinda!

#### Reduce Javascript that triggers reflows
- getBoundingClientRect, useful function, but everytime it is called it triggers a reflow to get information from the GPU
- We had a scrolling library that would determine position of what we were scrolling to with multiple getBoundingClientRect calls
- Library needs to be generic so it is a neccissary evil, but what if we build a scrolling library that fits our needs?
  - We know the size of our cells ahead of time, don't need to get the information from the browser
  - Calculate everything ahead of time and scroll to position using the calculation, avoiding even more draw calls! Awesome

#### Simplify the DOM
- React Virtualize
  - Some positions like FLX in NFL can have hundreds of players
  - React is fast but rendering that many components still slow, takes a lot of memory, and complicates the DOM tree
  - React Virtualize works a lot like RecyclerView from Android
  - Provides a list view, you provide a component for the list items
  - Only renders elements from the list currently in the view
  - Items outside the view are only rendered when brought into view
  - Turns rendering hundreds of players to only 6ish
- Remove unnecissary opacity calls
  - As mentioned, opacity can throw your component onto the compositor
  - Styles use a lot of opacity, but are on static backgrounds
  - Find the resulting color and set it explicitly
  




My recent task of adding a parallax effect to my site was an exercise in advanced Angular 17 capabilities and web performance best practices. The effect can be seen on the banner of my home page. This article shares an in-depth look at the technical implementation of this feature.

## Developing the Parallax Directive in Angular

The project's key component is a custom Angular 17 directive. This directive, designed to be dynamically reusable, accepts a 'ratio' input for varying the parallax depth. The directive interacts with a parallax service, which coordinates the visual effect based on scroll events.

### Window API Utilization

The implementation was based on two functions from the window browser API, `addEventListener` for listening for scroll events, and `requestAnimationFrame` to ensure performant animations.

The `addEventListener('scroll', ...)` method registers a listener for scroll events on the window object. It's crucial for triggering updates in response to user interaction, but using it directly for animations can lead to jank or unoptimized rendering cycles as it can be called many times per render frame; as I found out.

The `requestAnimationFrame()` method tells the browser that you wish to perform an animation and requests that the browser call a specified function to update an animation before the next repaint.

### Implementation Rationale

I had initially created an implementation without `requestAnimationFrame`. This caused a warning in Firefox related to potential performance issues. This warning wouldn't have been unique to Firefox, it was a signal to adopt better practices for smoother and more efficient animations. `requestAnimationFrame` ensures that the parallax effect is in sync with the browser's repaint cycle, providing a smoother visual experience and reducing the computational load.

### Integrating with Angular's SSR

Handling the parallax effect in Angular 17's Server-Side Rendering (SSR) environment presented another layer of complexity. Server-side environments lack access to browser-specific APIs like window. The implementation required checks to ensure these APIs are called only in a browser context, ensuring functionality on both client and server environments. Angular provides a helper function `isPlatformBrowser` to make this easy,

```js
export class ParallaxService {
    private platformId = inject(PLATFORM_ID);
    constructor() {
    if (isPlatformBrowser(this.platformId)) {
      window.addEventListener('scroll', this.onScroll.bind(this));
    }
  }
}
```

### Ensuring Performance

Performance was a key concern. Using the Firefox profiler, I confirmed that the parallax implementation did not affect the site's 60 fps target. The browser was able to consistently deliver a frame every 16ms-17ms.

_Parallax effect disabled._

_Parallax effect enabled._

## Complete Implementation

### Parallax Directive

The directive manages registration and real-time updates of elements with the parallax service.

```js
@Directive({
  selector: '[appParallax]',
  standalone: true,
})
export class ParallaxDirective implements OnInit, OnDestroy {
  @Input('ratio') parallaxRatio: number = 1;

  private eleRef = inject(ElementRef);
  private renderer = inject(Renderer2);
  private parallaxService = inject(ParallaxService);
  private updateFunction: (scrollPosition: number) => void;

  constructor() {
    this.updateFunction = (scrollPosition) => {
      const offset = scrollPosition * this.parallaxRatio;
      this.renderer.setStyle(
        this.eleRef.nativeElement,
        'transform',
        `translateY(${offset}px)`,
      );
    };
  }

  ngOnInit(): void {
    this.parallaxService.registerParallaxElement(this.updateFunction);
  }

  ngOnDestroy(): void {
    this.parallaxService.unregisterParallaxElement(this.updateFunction);
  }
}

```

### Parallax Service

This service listens for scroll events and uses `requestAnimationFrame` to update the position of parallax elements.

```js
@Injectable({
  providedIn: 'root',
})
export class ParallaxService {
  private scrollPosition = 0;
  private isScrolling = false;
  private parallaxElements: ((scrollPosition: number) => void)[] = [];
  private platformId = inject(PLATFORM_ID);

  constructor() {
    if (isPlatformBrowser(this.platformId)) {
      window.addEventListener('scroll', () => this.onScroll());
    }
  }

  registerParallaxElement(
    updateFunction: (scrollPosition: number) => void,
  ): void {
    this.parallaxElements.push(updateFunction);
  }

  unregisterParallaxElement(
    updateFunction: (scrollPosition: number) => void,
  ): void {
    this.parallaxElements = this.parallaxElements.filter(
      (func) => func !== updateFunction,
    );
  }

  private onScroll(): void {
    if (!isPlatformBrowser(this.platformId)) {
      return;
    }

    this.scrollPosition = window.scrollY;
    if (!this.isScrolling) {
      window.requestAnimationFrame(() => this.updateParallaxElements());
      this.isScrolling = true;
    }
  }

  private updateParallaxElements(): void {
    this.parallaxElements.forEach((updateFunction) =>
      updateFunction(this.scrollPosition),
    );
    this.isScrolling = false;
  }
}

```

### Usage

The directive can be added to any element, with a ratio to specify the element's scroll speed. This is how it was used to make the banner of the home page,

```html
<div class="absolute inset-0 z-[-1]">
  <div appParallax [ratio]="0.7" class="absolute inset-0 -top-1/2">
    <img
      src="/assets/image/welcome_banner/city_silhouette.png"
      alt="Background Image 1"
      class="w-full h-full object-cover"
    />
  </div>
  <div appParallax [ratio]="0.5" class="absolute inset-0 -top-1/2">
    <img
      src="/assets/image/welcome_banner/office_background.png"
      alt="Background Image 2"
      class="w-full h-full object-cover"
    />
  </div>
  <div appParallax [ratio]="0.3" class="absolute inset-0 left-1/2 top-20">
    <img
      src="/assets/image/welcome_banner/character.png"
      alt="Background Image 3"
      class="h-full w-auto object-cover"
    />
  </div>
</div>
```

## Final Thoughts

In closing, this project was a fantastic learning opportunity. I delved into Angular 17 and browser profilers, discovering how to make my website not only look more appealing but also perform better. I’m looking forward to applying these insights to future projects.

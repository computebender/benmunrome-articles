When I started out building my website, I knew I wanted something that didn't look like every other Bootstrap/Material app on the web, I wanted something more custom. This lead me to choose Tailwind CSS for it's ease of integration into my Angular project, and ease of use for creating truely custom interfaces.

## Why Tailwind?

Tailwind CSS stands apart from other popular CSS frameworks due to its utility-first approach. Unlike frameworks like Bootstrap that provide predefined components, Tailwind offers more granular control by allowing developers to apply utility classes directly to HTML elements. These utilities frquently map to a CSS property; but may also be more complex, adding multiple properties in one utility. In Tailwind CSS, styling a button involves applying a set of utilities for each style aspect. For example, a blue, rounded button with specific padding and text color is created using:

```html
<button class="bg-blue-500 text-white px-4 py-2 rounded">Click me</button>
```

This method offers detailed customization, unlike Bootstrap, where a single class creates a pre-designed button, as shown below:

```html
<button class="btn btn-primary">Click me</button>
```

Here, the Bootstrap code is shorter but offers far less customization. Often libraries such as Bootstrap or Material do provide methods of customizing the pre-defined styles, but only to a certain extent. Maintenance can also become an issue as the libraries evolve.

## Installing Tailwind in an Angular Project

The integration of Tailwind CSS into my Angular 17 project was simple, largely due to Tailwind's [installation guide](https://tailwindcss.com/docs/guides/angular), and initialization script. After installing necessary dependencies, I made minimal modifications to the `tailwind.config.js` to fit my project's needs:

```js
module.exports = {
  content: [
    "./src/**/*.{html,ts}",
  ],
  ...
  corePlugins: {
    preflight: false,
  }
}
```

Disabling preflight was a necessary step to maintain the styling of ngx-markdown (markdown to HTML parser) content. The preflight feature of Tailwind CSS removes all default styles, from all elements. This conflicted with ngx-markdown, removing styles from all markdown content.

## Tailwind in Action

### Layout Capabilities

Tailwind's responsive utilities played an important role in my project. The article grid is a prime example of how Tailwind makes responsive design easy:

```html
<div
  class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-x-4 gap-y-8"
>
  <!-- Articles rendering -->
</div>
```

Here, Tailwind's responsive classes are used to their full potential. The `grid-cols-1` class sets a single-column layout by default, while `md:grid-cols-2` changes the layout to two columns on medium-sized screens (`md:` prefix), and so on. This method of using prefixed classes like `md:` and `lg:` automatically applies styles based on screen size, eliminating the need for complex media queries.

### Styling Capabilities

The styling of individual article cards further shows off Tailwind's usefulness. In this example, each card uses Tailwind's styling classes for a clean, consistent look:

```html
<div class="p-6 border border-solid border-slate-600">
  <p class="font-bold font-sans">{{ article.title }}</p>
  <p>{{ article.summary }}</p>
  <p>{{ article.id }}</p>
  <a [routerLink]="['/blog', article.id, article.slug]">Article</a>
</div>
```

In this snippet, `p-6` adds padding; `border-solid` and `border-slate-600` defines the border style and color; `font-bold` and `font-sans` applies typography styles. These classes show how Tailwind's predefined styles can be combined to create distinct and appealing designs.

## Final Thoughts

Adopting Tailwind CSS for my personal website project was a game-changer. Its utility-first method provided an unbeatable level of control and simplicity in creating responsive and uniquely styled web pages. This method of inline styling turned out to be perfectly suited for my project's particular needs. For developers seeking a combination of simplicity, customization, and robust design capabilities, Tailwind CSS is a great choice!

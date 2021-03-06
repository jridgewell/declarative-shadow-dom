# Overview

This is a more elaborate example showing one possible way to use declarative
Shadow DOM for Server Side Rendering (SSR). In this example, there are two
custom elements, `<my-clock>` and `<my-carousel>`. The clock shows an animated world
clock for a given time zone, and with a configurable font size. The carousel
accepts any number of child elements, and displays them one at a time, with
controls to navigate backward and forward through the list.

Important note: I am not a UX designer, as will be clear when you see these
controls. However, the point of these examples is to use a few real controls
with real style changes and behaviors, and test their performance to see
how these elements could be coded.

Here is the code for the [clock](clock_element.js) and [carousel](carousel_element.js) elements. Each of these components can optionally use inline `<style>` elements for styling, or use a shared `adoptedStylesheets` approach to styling. These custom elements are both built to assume that any existing shadow root already contains the correct DOM tree. In this way, they can accept existing SSR versions of their content.

The [example page](https://mfreed7.github.io/declarative-shadow-dom/perf_tests/clock_example/example.html?styleType=inline) ([code here](example.html)) contains 30 `<my-carousel>` elements, each of which contains 35 `<my-clock>` elements. The clocks are set to different time zones and font sizes, and the carousels are set to different initially-selected elements. This page has GET options that select whether to use [inline](https://mfreed7.github.io/declarative-shadow-dom/perf_tests/clock_example/example.html?styleType=inline) styles or [adoptedStylesheets](https://mfreed7.github.io/declarative-shadow-dom/perf_tests/clock_example/example.html?styleType=adopted), and also an [argument](https://mfreed7.github.io/declarative-shadow-dom/perf_tests/clock_example/example.html?styleType=inline&includeSSRControls=true) that adds controls to make it easy to call the `getInnerHTML()` function to retrieve the SSR version of page content.

The [SSR version](https://mfreed7.github.io/declarative-shadow-dom/perf_tests/clock_example/example_SSR.html?styleType=inline) ([code here](example_SSR.html)) of the example page contains exactly the same Javascript code as the original example, plus it has a copy of the Server Side Rendered version of the HTML.


# File sizes

Here are the sizes of the four files listed above:

|     File | Size (bytes)   | Gzipped Size (bytes) | Compression Ratio |
|------------:|------------:|------------:|--------:|
| clock_element.js | 3,219  | 1,136 | 65% |
| carousel_element.js | 3,061  | 1,077 | 65% |
| example.html | 86,317  | 2,331 | 97% |
| example_SSR.html | 1,008,868  | 17,494 | 98% |

While the SSR version of the content is over 1MB, it compresses by 98% due to the repitition of components. So the final SSR payload is only about 18kB. And because no async JS resources are needed to render the SSR content, there are no further network round-trips needed to begin rendering.

# Results

Three test cases were run:

- Non-SSR example, with inline styles
- Non-SSR example, with adoptedStylesheets
- SSR example, with inline styles

All tests were performed on a fairly high-powered Lenovo P920 workstation, using Chrome 82.0.4068.4 on Linux, and with the `--enable-blink-features=DeclarativeShadowDOM` command line flag. Measurements were taken using [tachometer](https://www.npmjs.com/package/tachometer), and by measuring the time it took to get all content properly rendered on screen. In the non-SSR case, this was defined to be the point at which both the `<my-clock>` and `<my-carousel>` custom elements were defined. For the SSR case, this was defined to be the point at which the entire page had been parsed. In all cases, the example pages were served from [github.io](https://mfreed7.github.io/declarative-shadow-dom/perf_tests/clock_example/example.html?styleType=inline) over a relatively high-speed network.

## <a name="numerical_results"></a> Numerical Results

|     Version | <none>      |
|------------:|:------------|
|     Browser | chrome<br>82.0.4068.4 |
| Sample size | 250         |


| Benchmark           | Bytes      | Gzip Bytes   |            Avg time |    vs Inline Styles |   vs Adopted Styles | vs SSR (Inline Styles) |
|:--------------------|-----------:|-------------:|----------------------:|--------------------:|--------------------:|----------------------:|
| Inline Styles       | 92,674     |        4,568 | 267.84ms - 277.69ms |         | **slower**<br>4% - 10%<br>10.32ms - 24.98ms | **slower**<br>42% - 48%<br>79.40ms - 89.88ms |
| Adopted Styles      | 92,674     |        4,568 | 249.69ms - 260.54ms | **faster**<br>4% - 9%<br>10.32ms - 24.98ms |   | **slower**<br>32% - 39%<br>61.27ms - 72.70ms |
| SSR (Inline Styles) | 1,015,225  |       17,520 | 186.35ms - 189.91ms | **faster**<br>30% - 32%<br>79.4ms - 89.88ms | **faster**<br>25% - 28%<br>61.27ms - 72.7ms |  |



## Qualitative Results

One thing that becomes obvious when loading the sample pages is that the non-SSR versions of the example show a flash of unstyled (or partially styled) content. As the examples are written, the two script elements are written like this:

```html
<script type="module" src="carousel_element.js"></script>
<script type="module" src="clock_element.js"></script>
```

so that the `<my-carousel>` element is defined first. This leads to the carousel elements being upgraded first, before the clocks, and this leads to a first render of a column of empty carousels. Then, the `<my-clock>` elements upgrade, and the final rendered output looks correct.

Very importantly, if the order of these two script tags above is reversed, so that the clock elements are upgraded first, the non-SSR performance shown in the [Numerical Results](#numerical_results) section is about 2X worse. This is because all of the clocks get rendered first, before the carousels have a chance to upgrade and hide all but one clock per carousel. A page-level CSS rule like this was added to minimize the effect of the ordering:

```css
my-clock:not(:defined), my-carousel:not(:defined) {
  display: none;
}
```

With this rule in place, the order of the element scripts no longer changes the results appreciably.

# Conclusions

Between the two non-SSR examples, the version using adoptedStylesheets is about 4.5% faster than the version using inline styles. However, the SSR example is about 30% faster than both of the non-SSR examples, despite the HTML content being over 10x larger. Additionally, the SSR version does not suffer from a flash of partially-styled content.

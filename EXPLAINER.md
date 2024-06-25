# Explainer of the μLottie project

μLottie is a dedicated compiler that focuses on making vector animations lightweight again.

The goal is to take a LAC(Lottie Animation Community) [spec](https://lottie.github.io/lottie-spec/) compliant file as input and generate an optimized JavaScript/DOM code without heavy additional runtime.

## Background

Our mobile app, Karrot, has a variety of services integrated through a WebView. It is a kind of "micro front-end" configuration, but because the runtime is not shared across different domains, making the runtime lightweight is a serious challenge for our mobile experiences.

## The problem

We use Lottie for improving visuals, like adding micro interactions, effects, puting beatiful animations while wating something.

Lottie animation has quickly gained popularity for its lighter and more accurate expressions than GIFs. Designers can still use the most familiar tool, Adobe After Effects.

However, the few years of using Lottie have not been happy times for Web developers.

Airbnb's [lottie-web](https://github.com/airbnb/lottie-web), the most popular lottie player on the web today, is almost 300 kB (75 kB gzipped), which is more than twice the size of react + react-dom which are our most important dependencies. There is a "light mode" with subsetting capabilities, but it is still 170 kB (60 kB gzipped) in size.

[jLottie](https://github.com/lottiefiles/jlottie), a lightweight player developed by LottieFiles, has officially discontinued development.

Their brand new [lottie player](https://github.com/LottieFiles/lottie-player) is even heavier, at 369 kB (94 kB gzipped) and includes a WebAssembly binary that is almost 1 MB (400 kB gzipped) in size. This is unacceptable to us.

The animation displayed during loading takes up more time than the actual resource being loaded. The runtime required to render the animation takes even more times than the animation itself. This adds a ridiculous waterfall effect.

### Why Lottie players are so heavy?

Simply because Lottie and After Effects has so many features. Lottie's goal is to provide as many After Effects features as possible across multiple platforms, that's not a trivial task.

Lottie was originally created to support mobile apps, and size of the player was not a big deal for them due to they are installed onto devices.

### dotLottie has a different goal

[dotLottie](https://dotlottie.io/) (`.lottie`) is a format being developed by LottieFiles. It is a ZIP-based container that can bundle multiple Lottie animations at once.

It's the exact opposite of the Web's default resource loading behavior, which is to only partially load what's needed.

So dotLottie is only useful if the app actively utilizes many animations and can preload them in advance. Particulary, our app doesn't use animations on all screens, nor does it have any image resources associated with it, so that's only a minus.

They claim that deflate applied to ZIP reduces file size, but It's not really useful with little animation. Instead using Web's default deflate/gzip compression infrastructures, using custom decoder & decompressor adds significant overhead on the main UI thread and possibly jank.

## Design concepts

"Renderer" will never be lightweight.

Also, unlike rendering, the bottleneck of the Lottie format itself or resource loading has not yet been studied much. Its JSON-based specification has fundamental limitations.

Of course, there would be much more optimized tools like [Rive](https://rive.app), but we don't want to change the entire workflow. Instead, we're trying to eliminate inefficiencies in advance through the compiler.

### AOT compliation

μLottie is not a runtime library but an AOT(Ahead-of-Time) compiler. It analyzes the animation requirements in advance and outputs independently executable JavaScript runtime code in a module format. This avoids bloating from unused features and achieves additional optimizations by leveraging JavaScript’s excellent bundler toolchains.

### Pre-rederable content

μLottie animations are considered a [critical path](https://web.dev/learn/performance/understanding-the-critical-path). This means they are rendered immediately without any waiting as the screen transitions.

μLottie specifies the initial frame and represents the rest of the animation using deltas. This approach allows it to display visuals without additional overhead when pre-rendering is required, such as on the server-side rendering or with `<noscript>`.

### Pre-loadable assets

Lottie can includes images within the animation. Images are embedded as data URIs in general, dotLottie made it as file references.

μLottie choose both strategies. Assets smaller than a certain threshold are used as data URIs, while larger assets are converted into file references to be loaded concurrently. μLottie provides a manifest to optimize priority using [early hints](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/103) on the web server.

### Trade-offs

μLottie generates code for each animation, which can lead to redundant code. This is fine mostly when using different animations for each path, but if multiple animations need to be shown consecutively within a single path or a segment, it might be better to use an optimized runtime library for Lottie.

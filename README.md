# jest-transform-styl

A Jest transformer which enables importing Stylus into Jest's `jsdom`. If you are looking for just Jest CSS transformer, use `jest-transform-css` or use this package without Stylus setting. Details below.

**If you are not here for Visual Regression Testing, but just want to make your tests work with CSS Modules, then you are likley looking for https://github.com/keyanzhang/identity-obj-proxy/.**

## Description

When you want to do Visual Regression Testing in Jest, it is important that the styles used components is available to the test setup. So far, CSS was not part of tests as it was mocked away by using `moduleNameMapper` like a file-mock or `identity-obj-proxy`.  Package `jest-transform-css` is a great starting point if your app is using CSS or CSS modules. However, if your app is using stylus, you may need additional stylus transformer for Jest.

`jest-transform-styl` is forked from `jest-transform-css` with additional transformer feature for stylus. The package is intended to be used in an `jsdom` environment. When any component imports CSS in the test environment, then the loaded CSS will get added to `jsdom` using [`style-inject`](https://github.com/egoist/style-inject) - just like the Webpack CSS loader would do in a production environment. This means the full styles are added to `jsdom`.

This doesn't make much sense at first, as `jsdom` is headless (non-visual). However, we can copy the resulting document markup ("the HTML") of `jsdom` and copy it to a [`puppeteer`](https://github.com/googlechrome/puppeteer/) instance. We can let the markup render there and take a screenshot there. The [`jsdom-screenshot`](https://github.com/dferber90/jsdom-screenshot) package does exactly this.

Once we obtained a screenshot, we can compare it to the last version of that screenshot we took, and make tests fail in case they did. The [`jest-image-snapshot`](https://github.com/americanexpress/jest-image-snapshot) plugin does that.

## Setup

### Installation

```bash
npm install jest-transform-styl --save-dev
```

The old setup of CSS in jest needs to be removed, and the new setup needs to be added next.

### Removing module name mapping

If your project is using plain CSS imported in the components, then you're likely using a mock file. You can remove that configuration.

```diff
// in the Jest config
"moduleNameMapper": {
- "\\.(styl|css|less)$": "<rootDir>/__mocks__/styleMock.js"
},
```

If your project is using CSS Modules, then it's likely that `identity-obj-proxy` is configured. It needs to be removed in order for the styles of the `jest-transform-css` to apply.

So, remove these lines from `jest.config.js`:

```diff
// in the Jest config
"moduleNameMapper": {
-  "\\.(s?css|less)$": "identity-obj-proxy"
},
```

### Adding `transform`

Open `jest.config.js` and modify the `transform`:

```
// in the Jest config
transform: {
  "^.+\\.js$": "babel-jest",
  "^.+\\.css$": "jest-transform-css",
  "^.+\\.styl$": "jest-transform-styl"
}
```

> Notice that `babel-jest` gets added as well.
>
> The `babel-jest` code preprocessor is enabled by default, when no other preprocessors are added. As `jest-transform-css` and `jest-transform-styl` are code preprocessors, `babel-jest` gets disabled when `jest-transform-css` and `jest-transform-styl` are added.
>
> So it needs to be added again manually.
>
> See https://github.com/facebook/jest/tree/master/packages/babel-jest#setup

### Enabling Stylus

This package supports CSS, CSS Modules and Stylus. If you are not using Stylus, no need to provide `stylus: true` setting.

If you are using both modules and stylus, follow steps below:

```js
// jesttransformcss.config.js

module.exports = {
  modules: true,
  stylus: true,
  pathToStyles: 'src/styles'  // this is the path where your styles reside. used for resolving path provided by `composes` keyword in stylus
};
```

  -  `modules`: This will enable CSS module transformation for all CSS and Stylus files transformed by `jest-transform-styl`.
  - `stylus`: This tells the transformer that the application uses stylus files and it needs to transform stylus to CSS before doing rest of CSS transformation for Jest.
  - `pathToStyles` is an optional path to styles which points to your styles file. This path is used to compile the keywords like     `composes` used in Stylus files to inherit styles from other stylus files. 


If your setup uses both, CSS modules and regular CSS, then you can determine how to load each file individually by specifying a function:

```js
// jesttransformcss.config.js

module.exports = {
  modules: filename => filename.endsWith(".mod.css")
};
```

This will load all files with `.mod.css` as CSS modules and load all other files as regular CSS. Notice that the function will only be called for whichever regex you provided in the `transform` option of the Jest config.


**Note**: *If your stylus files are using `require` to import styles from other stylus files, you will run into some issues since stylus cannot compile a relative path. You either will have to provide an absolute path of the file or will need to change the path into a webpack alias, eg. ~styles, and regex the `~` with the root path. This change will have to be done in this package itself.*


### PostCSS

If your setup is using `PostCSS` then you should add a `postcss.config.js` at the root of your folder.

You can apply certain plugins only when `process.env.NODE_ENV === 'test'`. Ensure that valid CSS can be generated.

> `jest-transform-css` is likley not flexible enough yet to support more sophisticated PostCSS configurations. However, we should be able to add this functionality by extending the configuration file. Feel free to open an issue with your setup and we'll try to support it.

### css-loader

If your setup is using `css-loader` only, without PostCSS then you should be fine.
If you have `modules: true` enabled in `css-loader`, you need to also enable it for `jest-transform-css` (see "Enabling CSS modules"). When components import CSS modules in the test environment, then the CSS is transformed through PostCSS's `cssModules` plugin to generate the classnames. It also injects the styles into `jsdom`.

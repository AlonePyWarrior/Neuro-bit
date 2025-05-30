#### َAuthor: Ali Esmaeilzadeh<br><small>a guide for using Webpack in Django project</small>

Using Webpack to optimize static resources (like DaisyUI, Tailwind, and other CSS/JS files) in a Django project is a powerful way to improve performance and ensure a well-structured project. Below are step-by-step instructions on how to set up Webpack in your Django project for optimizing static resources.

### Step 1: Install Node.js and NPM
First, ensure you have [Node.js](https://nodejs.org/) and [NPM](https://www.npmjs.com/) installed on your machine. These are required to use Webpack.

To verify installation, run:

```bash
node -v
npm -v
```

### Step 2: Initialize Node.js Project
In the root of your Django project, initialize a `package.json` file, which will manage the dependencies for Webpack and other frontend tools:

```bash
npm init -y
```

### Step 3: Install Webpack and Required Plugins
Install Webpack, Webpack CLI, and any other required plugins like `css-loader`, `style-loader`, and `mini-css-extract-plugin` for handling CSS, along with PostCSS to handle TailwindCSS:

```bash
npm install webpack webpack-cli css-loader style-loader mini-css-extract-plugin postcss postcss-loader tailwindcss autoprefixer --save-dev
```

If you're using DaisyUI, install it as well:

```bash
npm install daisyui
```

### Step 4: Setup Tailwind and PostCSS Configuration
Create a `postcss.config.js` file in the root of your project to configure Tailwind:

```js
module.exports = {
  plugins: [
    require('tailwindcss'),
    require('autoprefixer'),
  ]
}
```

Then create a `tailwind.config.js` file to configure Tailwind CSS (including DaisyUI):

```js
module.exports = {
  content: [
    './templates/**/*.html',
    './static/**/*.js',
  ],
  theme: {
    extend: {},
  },
  plugins: [require('daisyui')],
}
```

### Step 5: Configure Webpack
Create a `webpack.config.js` file in the root of your project. This file will configure Webpack to handle your static assets like CSS and JS.

```js
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  mode: 'production',
  entry: './static/src/index.js', // Your entry point for JS/CSS
  output: {
    path: path.resolve(__dirname, './static/dist'), // Output directory
    filename: 'bundle.js', // Output JS file
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader', // Use PostCSS for Tailwind
        ],
      },
    ],
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: 'styles.css', // Output CSS file
    }),
  ],
};
```

### Step 6: Create Entry Point Files
In your Django project’s `static/` folder, create a directory structure like `src/` for source files and `dist/` for Webpack output:

```
static/
  src/
    index.js
    styles.css
  dist/
    (Webpack output)
```

**Example `index.js`:**

```js
import './styles.css'; // This will include Tailwind and other CSS
```

**Example `styles.css`:**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Add any custom styles here */
```

### Step 7: Update Django Settings
In `settings.py`, add or update your `STATICFILES_DIRS` to include the Webpack output:

```python
STATICFILES_DIRS = [
    BASE_DIR / "static",
    BASE_DIR / "static/dist",  # Add Webpack output directory here
]
```

### Step 8: Run Webpack
To build your assets using Webpack, run:

```bash
npx webpack --config webpack.config.js
```

This will bundle your JS and CSS and output it into the `static/dist` directory.

### Step 9: Use Bundled Assets in Django Templates
In your Django templates, include the bundled CSS and JS files generated by Webpack:

```html
<link rel="stylesheet" href="{% static 'dist/styles.css' %}">
<script src="{% static 'dist/bundle.js' %}"></script>
```

### Step 10: Set Up Webpack for Development and Production
You can adjust Webpack for both development and production environments. For example, in development, you might want to enable source maps and watch for changes.

**Add these settings in `webpack.config.js`:**

```js
mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
devtool: process.env.NODE_ENV === 'production' ? false : 'source-map',
watch: process.env.NODE_ENV === 'production' ? false : true,
```

### Step 11: Automate Builds with NPM Scripts
To make it easier to run Webpack, you can add scripts to your `package.json`:

```json
"scripts": {
  "build": "webpack --config webpack.config.js",
  "watch": "webpack --watch --config webpack.config.js"
}
```

Now, you can run `npm run build` to bundle your assets or `npm run watch` to automatically rebuild on file changes.

---

### Summary
1. Install Node.js and Webpack dependencies.
2. Set up Tailwind and DaisyUI with PostCSS.
3. Configure Webpack to bundle and optimize your static resources.
4. Use Webpack output in your Django templates.
5. Automate your build process using NPM scripts.

This setup will optimize your CSS (Tailwind, DaisyUI) and JS, ensuring efficient use of static resources in your Django project.

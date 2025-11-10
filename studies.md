# devDependencies

## module-esd package.json

```json
{
  "name": "1t21-aura-module-esd",
  "version": "0.0.7",
  "scripts": {
    "start:dev": "webpack serve --port 9018",
    "start:standalone": "webpack serve --env standalone --port 9018",
    "start": "webpack serve --port 3000 --env goal=$npm_config_env",
    "build": "webpack --mode=production",
    "build:dev": "webpack --mode=production --env goal=development",
    "build:val": "webpack --mode=production --env goal=validation",
    "build:prod": "webpack --mode=production --env goal=production",
    "build:webpack": "webpack --mode=production",
    "analyze": "webpack --mode=production --env analyze",
    "lint": "eslint",
    "lint:fix": "eslint --fix",
    "test": "cross-env BABEL_ENV=test jest",
    "watch-tests": "cross-env BABEL_ENV=test jest --watch",
    "prepare": "node .husky/install.mjs",
    "coverage": "cross-env BABEL_ENV=test jest --coverage",
    "build:types": "tsc",
    "cdk:synth:dev": "npx cdk synth -v --context stage=dev",
    "cdk:diff:dev": "npx cdk diff -v --context stage=dev",
    "cdk:deploy:dev": "npx cdk deploy -v --require-approval never --context stage=dev",
    "cdk:synth:val": "cdk synth -v --context stage=val",
    "cdk:diff:val": "cdk diff -v --context stage=val",
    "cdk:deploy:val": "cdk deploy -v --require-approval never --context stage=val",
    "cdk:synth:prod": "cdk synth -v --context stage=prod",
    "cdk:diff:prod": "cdk diff -v --context stage=prod",
    "cdk:deploy:prod": "cdk deploy -v --require-approval never --context stage=prod",
    "cdk:synth:check": "rm -f output.yaml && cdk synth --context stage=dev -v >> output.yaml"
  },
"devDependencies": {
    "@babel/core": "^7.25.2",
    "@babel/eslint-parser": "^7.25.1",
    "@babel/plugin-transform-runtime": "^7.24.7",
    "@babel/preset-env": "^7.25.3",
    "@babel/preset-react": "^7.24.7 ",
    "@babel/preset-typescript": "^7.23.3",
    "@babel/runtime": "^7.25.0",
    "@eslint/js": "^9.17.0",
    "@testing-library/jest-dom": "^6.x",
    "@testing-library/react": "^14.x",
    "@types/jest": "^29.5.12",
    "@types/react": ">=18.0.28 <=18.2.42",
    "@types/react-dom": "^18.2.22",
    "@types/systemjs": "^6.13.5",
    "@types/webpack-env": "^1.18.5",
    "babel-jest": "^29.7.0",
    "concurrently": "^8.2.2",
    "cross-env": "^7.0.3",
    "dotenv-webpack": "^8.1.0",
    "eslint": "^9.17.0",
    "eslint-plugin-react": "^7.37.3",
    "globals": "^15.14.0",
    "husky": "^9.1.4",
    "identity-obj-proxy": "^3.0.0",
    "jest": "^29.7.0",
    "jest-cli": "^29.7.0",
    "jest-environment-jsdom": "^29.7.0",
    "react-test-renderer": "^18.3.1",
    "ts-config-single-spa": "^3.0.0",
    "typescript": "^5.5.4",
    "typescript-eslint": "^8.19.0",
    "webpack": "^5.93.0",
    "webpack-cli": "^5.1.4",
    "webpack-config-single-spa-react": "^4.0.5",
    "webpack-config-single-spa-react-ts": "^4.0.5",
    "webpack-config-single-spa-ts": "^4.1.4",
    "webpack-dev-server": "^5.2.2",
    "webpack-merge": "^5.8.0"
  },
  "dependencies": {
    "@airbus/components-react": "^3.1.0",
    "@airbus/icons": "^3.0.3",
    "@emotion/react": "^11.14.0",
    "@emotion/styled": "^11.14.1",
    "@fontsource/inter": "^5.2.6",
    "@mui/icons-material": "^7.3.1",
    "@mui/material": "^7.3.1",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "single-spa": "^6.0.1",
    "single-spa-react": "^6.0.1"
  },
```

## probably global tsconfig.json

```json
{
  "extends": "ts-config-single-spa",
  "compilerOptions": {
    "jsx": "react-jsx",
    "declarationDir": "dist"
  },
  "files": [
    "src/CoreElec-aura-module-sidebar.tsx"
  ],
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "src/**/*.test*"
  ]
}
```

## probably global babel.config.json

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": "current node"
      }
    ],
    [
      "@babel/preset-react",
      {
        "runtime": "automatic"
      }
    ],
    "@babel/preset-typescript"
  ],
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "useESModules": true,
        "regenerator": false
      }
    ]
  ],
  "env": {
    "test": {
      "presets": [
        [
          "@babel/preset-env",
          {
            "targets": "current node"
          }
        ]
      ]
    }
  }
}
```

### sidebar webpack.config.js

```javascript
/* eslint-disable indent */
const ORG_NAME = "CoreElec"; // Don't Modify it 
const PROJECT_NAME = "aura-module-sidebar"; // (used for folder and entry file)
const ROUTE_NAME = "sidebar"; // (used in 1T21-aura-orchestrator <route path="..." -> Mandatory to work with Module federation)
const PREFIX = "aura"; // Don't Modify it must match same value as the Orchestrator (used for publicPath and devServer)

const { merge } = require("webpack-merge");
const singleSpaDefaults = require("webpack-config-single-spa-ts");
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const webpack = require("webpack");
const TerserPlugin = require("terser-webpack-plugin");
const Dotenv = require('dotenv-webpack');

module.exports = (webpackConfigEnv, argv) => {
  const defaultConfig = singleSpaDefaults({
    orgName: ORG_NAME,
    projectName: PROJECT_NAME,
    webpackConfigEnv,
    argv,
  });

  let currentPath = '';
  let envPath = '';
  const configEnv = webpackConfigEnv.goal

  switch (configEnv) {
    case 'development':
      currentPath = `https://dev.coreelec.1t21-coreelec.aws.cloud.airbus-v.corp/${PREFIX}`
      envPath = 'environment.dev.ts'
      break;
    case 'validation':
      currentPath = `https://val.coreelec.1t21-coreelec.aws.cloud.airbus-v.corp/${PREFIX}`
      envPath = 'environment.val.ts'
      break;
    case 'production':
      currentPath = `https://coreelec.1t21-coreelec.aws.cloud.airbus.corp/${PREFIX}`
      envPath = 'environment.prod.ts'
      break;
    default:
      currentPath = ''
      envPath = 'environment.ts'
  }

  return merge(defaultConfig, {
    externals: {
      'single-spa': 'single-spa',
      'single-spa-layout': 'single-spa-layout'
    },
    output: {
      publicPath: currentPath
    },
    optimization: {
      minimize: true,
      minimizer: [new TerserPlugin()],
    },
    devServer: {
      setupMiddlewares: (middlewares, devServer) => {
        if (!devServer) {
          throw new Error('webpack-dev-server is not defined');
        }
        middlewares.unshift({
          name: 'first-in-array',
          path: `/${PREFIX}/status`,
          middleware: (req, res) => {
            res.json("running");
          },
        });
        return middlewares;
      }
    },
    plugins: [
      // "NODE_ENV" have to be imported by using Airbus Design System
      // .env file is not needed as already share by webpack script
      new Dotenv(),
      new webpack.NormalModuleReplacementPlugin(
        /environments\/environment\.ts/,
        envPath
      ),
      new ModuleFederationPlugin({
        // ROUTE_NAME should match the <route path="..."> in ce-root-config
        name: ROUTE_NAME,
        library: { type: 'var', name: ROUTE_NAME },
        filename: `${ORG_NAME}-${PROJECT_NAME}.js`,
        shared: [
          { "react": { singleton: true, eager: true, requiredVersion: '^18' } },
          { "react-dom": { singleton: true, eager: true, requiredVersion: '^18' } },
          { "@airbus/components-react": { singleton: true, eager: true, requiredVersion: '^3' } },
          { "@airbus/icons": { singleton: true, eager: true, requiredVersion: '^3' } },
          { "@fontsource/inter": { singleton: true, eager: true, requiredVersion: '^5' } },
        ]
      }),
    ],
  });
};

```

### orchestrator webpack.config.js

```javascript


/**
 * Webpack configuration for building the root-config bundle.
 *
 * - The `output.filename` is set to 'static/root-config.js', which means the built JavaScript file
 *   will be placed inside a 'static' directory within the output path. This helps organize static assets
 *   (like JavaScript, CSS, images) under a dedicated folder, making it easier to manage and serve them.
 * - The `publicPath` is set dynamically using the `PREFIX` environment variable, ensuring that all assets
 *   are referenced from the correct base URL, especially when deploying under different subpaths.
 * - The configuration also includes plugins and loaders for copying assets, HTML generation, CSS minimization,
 *   and Babel transpilation.
 *
 * @module webpack.config
 */

const CopyWebpackPlugin = require('copy-webpack-plugin');
const path = require('path');
const fs = require('fs');
const webpack = require('webpack');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

// Get product and environment (local, dev, val, prod)

const PRODUCT = process.env.PRODUCT;
const ENV = process.env.ENV || 'local';
const PREFIX = process.env.PREFIX || 'aura';

if (!PRODUCT) {
  throw new Error(
    'Please specify the PRODUCT environment variable (e.g., set PRODUCT=a320 && npm run build:root:prod)'
  );
}

let localProxy = [
  {
    context: ['/import'],
    target:
      'https://dev-v2.coreelec.1t21-coreelec.aws.cloud.airbus-v.corp/aura/static',
    changeOrigin: true,
    secure: false,
    ws: false,
    pathRewrite: { '^/import': '' },
  },
  {
    context: ['/V1import'],
    target: 'https://dev.coreelec.1t21-coreelec.aws.cloud.airbus-v.corp',
    changeOrigin: true,
    secure: false,
    ws: false,
    pathRewrite: { '^/V1import': '' },
  },
];

if (ENV != 'local') {
  localProxy = [];
}

let webSocketURL;
if (ENV === 'local') {
  webSocketURL = 'wss://localhost:9000';
} else if (ENV === 'dev') {
  webSocketURL = 'wss://dev.core-elec.airbus.corp';
} else {
  webSocketURL = 'wss://localhost:9000';
}

module.exports = {
  mode: 'production',
  entry: './src/root-config.js',
  output: {
    path: path.resolve(__dirname, 'dist', PREFIX),
    filename: 'static/root-config.js',
    publicPath: `/${PREFIX}/`, // Ensure assets are always loaded from the correct prefix
    clean: true, // Clean the dist folder before each build
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
        },
      },
      {
        test: /\.css$/,
        use: [
          {
            loader: 'style-loader',
            options: { injectType: 'singletonStyleTag' },
          },
          'css-loader',
        ],
      },
    ],
  },
  plugins: [
    // Set PRODUCT to be avalable inside root-config.js
    new webpack.DefinePlugin({
      PRODUCTVAR: JSON.stringify(PRODUCT),
      ENVVAR: JSON.stringify(ENV),
    }),
    // Make favicon available
    new CopyWebpackPlugin({
      patterns: [
        {
          from: path.resolve(__dirname, 'src/favicon.ico'),
          to: path.resolve(__dirname, 'dist', PREFIX, 'static', 'favicon.ico'),
        },
      ],
    }),
    (() => {
      // 1. Read the product's import map for the current environment
      const importMapPath = `./products/${PRODUCT}/importmaps/importmap.${ENV}.json`;
      let importMap;
      try {
        importMap = require(importMapPath);
      } catch (e) {
        throw new Error(`Missing import map: ${importMapPath}`, e);
      }

      // 2. Read the product's dedicated layout file
      const layout = fs.readFileSync(
        path.resolve(__dirname, `./products/${PRODUCT}/layout.html`),
        'utf-8'
      );

      const finalImportMap = {
        imports: {
          'single-spa':
            'https://cdn.jsdelivr.net/npm/single-spa@6.0.3/lib/es2015/system/single-spa.min.js',
          'single-spa-layout':
            'https://cdn.jsdelivr.net/npm/single-spa-layout@2.1.0/dist/system/single-spa-layout.min.js',
          'react@18': 'https://unpkg.com/react@18/umd/react.production.min.js',
          'react-dom@18':
            'https://unpkg.com/react-dom@18/umd/react-dom.production.min.js',
          ...importMap.imports,
        },
      };

      return new HtmlWebpackPlugin({
        filename: 'index.html',
        template: './src/index.ejs',
        templateParameters: {
          importMap: JSON.stringify(finalImportMap, null, 2),
          // Pass the layout HTML as a parameter
          layout: layout,
          productName: PRODUCT,
          baseUrl: PREFIX,
          staticPath: `${PREFIX}/static`,
          productTitle: `Core Elec - ${PRODUCT.toUpperCase()}`,
        },
      });
    })(),
    new HtmlWebpackPlugin({
      filename: 'auth.html',
      template: './src/auth.html',
    }),
  ],
  optimization: {
    minimize: true,
    splitChunks: false,
    runtimeChunk: false,
    minimizer: ['...', new CssMinimizerPlugin()],
  },
  devServer: {
    ...(ENV === 'local'
      ? {
        server: {
          type: 'https',
          options: {
            key: fs.readFileSync(path.join(__dirname, 'secret', 'local.key')),
            cert: fs.readFileSync(path.join(__dirname, 'secret', 'local.pem')),
          },
        },
        client: { reconnect: 0, webSocketURL },
      }
      : {}),
    static: {
      directory: path.resolve(__dirname, 'dist', PREFIX),
      publicPath: `/${PREFIX}/`,
      watch: true,
    },
    port: 9000,
    historyApiFallback: {
      disableDotRule: true,
      index: `/${PREFIX}/index.html`,
      rewrites: [
        { from: new RegExp(`^/${PREFIX}(/.*)?$`), to: `/${PREFIX}/index.html` },
      ],
    },
    proxy: localProxy,
  },
  externals: {
    react: 'react@18',
    'react-dom': 'react-dom@18',
  },
};


```

### orchestrator package.json

```json

{
  "name": "1t21-aura-orchestrator",
  "version": "0.5.1",
  "main": "index.js",
  "scripts": {
    "start": "npm run start:a320:local",
    "build": "npm run build:a320:prod",
    "build:dev": "npm run build:a320:dev",
    "build:val": "npm run build:a320:val",
    "lint": "eslint \"{src,apps,libs,test}/**/*.js\"",
    "lint-fix": "eslint \"{src,apps,libs,test}/**/*.js\" --fix",
    "start:a320:local": "cross-env PRODUCT=a320 ENV=local webpack-dev-server --open /aura/",
    "start:a380:local": "cross-env PRODUCT=a380 ENV=local webpack-dev-server --open /aura/",
    "build:a320:local": "cross-env PRODUCT=a320 ENV=local webpack",
    "build:a320:dev": "cross-env PRODUCT=a320 ENV=dev webpack",
    "build:a320:val": "cross-env PRODUCT=a320 ENV=val webpack",
    "build:a320:prod": "cross-env PRODUCT=a320 ENV=prod webpack",
    "build:a380:local": "cross-env PRODUCT=a380 ENV=local webpack",
    "build:a380:dev": "cross-env PRODUCT=a380 ENV=dev webpack",
    "build:a380:val": "cross-env PRODUCT=a380 ENV=val webpack",
    "build:a380:prod": "cross-env PRODUCT=a380 ENV=prod webpack",
    "cdk:synth:dev": "npx cdk synth -v --context stage=dev",
    "cdk:diff:dev": "npx cdk diff -v --context stage=dev",
    "cdk:deploy:dev": "npx cdk deploy -v --require-approval never --context stage=dev",
    "cdk:synth:val": "cdk synth -v --context stage=val",
    "cdk:diff:val": "cdk diff -v --context stage=val",
    "cdk:deploy:val": "cdk deploy -v --require-approval never --context stage=val",
    "cdk:synth:prod": "cdk synth -v --context stage=prod",
    "cdk:diff:prod": "cdk diff -v --context stage=prod",
    "cdk:deploy:prod": "cdk deploy -v --require-approval never --context stage=prod",
    "cdk:synth:check": "rm -f output.yaml && cdk synth --context stage=dev -v >> output.yaml"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "description": "",
  "dependencies": {
    "@airbus/1t21-aura-library-notification-manager-core": "^0.0.0",
    "@airbus/1t21-aura-library-notification-manager-toast-manager": "^0.0.0",
    "@airbus/1t21-aura-library-permission-manager-core": "0.1.19",
    "@airbus/1t21-authentication-manager": "0.2.3",
    "single-spa": "^6.0.3",
    "single-spa-layout": "^3.0.0",
    "toastify-js": "^1.12.0"
  },
  "devDependencies": {
    "@airbus/1t21-ce-cdk-node-lib": "0.0.5",
    "@babel/core": "^7.28.0",
    "@babel/eslint-parser": "^7.28.0",
    "@babel/plugin-transform-runtime": "^7.28.3",
    "@babel/preset-env": "^7.28.0",
    "@babel/runtime": "^7.28.4",
    "@eslint/js": "^9.34.0",
    "babel-loader": "^10.0.0",
    "copy-webpack-plugin": "^13.0.0",
    "cross-env": "^7.0.3",
    "css-loader": "^7.1.2",
    "css-minimizer-webpack-plugin": "^7.0.2",
    "eslint": "^9.34.0",
    "eslint-config-prettier": "^10.1.5",
    "eslint-plugin-react": "^7.37.5",
    "globals": "^16.3.0",
    "html-webpack-plugin": "^5.6.3",
    "prettier": "^3.6.2",
    "style-loader": "^4.0.0",
    "typescript-eslint": "^8.42.0",
    "webpack": "^5.99.9",
    "webpack-cli": "^6.0.1",
    "webpack-dev-server": "^5.2.2"
  }
}


```

## orchestrator product importmaps

### a320 importmap.local.json

```json
{
  "imports": {
    "@CoreElec/data-exposure-module": "/V1import/ce-cockpit-data-exposure/CoreElec-data-exposure-module.js",
    "@CoreElec/aura-module-sidebar": "/import/aura-module-sidebar/CoreElec-aura-module-sidebar.js",
    "@CoreElec/aura-module-topbar": "/import/aura-module-topbar/CoreElec-aura-module-topbar.js",
    "@CoreElec/aura-module-default-content": "/import/aura-module-default-content/CoreElec-aura-module-default-content.js",
    "@CoreElec/aura-module-esd": "/import/aura-module-esd/CoreElec-aura-module-esd.js"
  }
}
```

### a320 importmap.dev.json

```json

{
  "imports": {
    "@CoreElec/data-exposure-module": ....-cockpit-data-exposure/CoreElec-data-exposure-module.js",
    "@CoreElec/aura-module-sidebar": ..../aura/static/aura-module-sidebar/CoreElec-aura-module-sidebar.js",
    "@CoreElec/aura-module-topbar": ..../aura/static/aura-module-topbar/CoreElec-aura-module-topbar.js",
    "@CoreElec/aura-module-esd": ..../aura/static/aura-module-esd/CoreElec-aura-module-esd.js",
    "@CoreElec/aura-module-default-content": ..../aura/static/aura-module-default-content/CoreElec-aura-module-default-content.js"
  }
}

```

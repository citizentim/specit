# specit

> specit is a fork of [speculate](https://github.com/bbc/specit) with some different goals.

Automatically generates an RPM Spec file for your Node.js project

## Installation

```
npm install --global specit
```

## Features

* Generates an RPM Spec file for your project
* (optional) Creates a [systemd](https://www.freedesktop.org/wiki/Software/systemd/) service definition file
* Supports configuration using your existing `package.json`

## Usage

Let's start with a simple Node.js project:

```
my-cool-api
├── package.json
└── server.js

0 directories, 2 files
```

First run npm install to install your dependencies:

```
npm install
```

This creates the `node_modules` directory:

```
my-cool-api
├── node_modules
├── package.json
└── server.js

1 directory, 2 files
```

Run the `specit` command from inside the project directory:

```
specit
```

You've now got an RPM Spec file and a systemd service definition for your project. You'll also notice that your application has been packaged into a `tar.gz` archive, ready to be built with an RPM building tool like [`rpmbuild`](http://www.rpm.org/max-rpm-snapshot/rpmbuild.8.html) or [`mock`](https://fedoraproject.org/wiki/Mock):

```
my-cool-api
├── SOURCES
│   └── my-cool-api.tar.gz
├── SPECS
│   └── my-cool-api.spec
├── node_modules
├── my-cool-api.service
├── package.json
└── server.js

3 directories, 5 files
```

Specit is designed to be used at build time, just before you package your application into an RPM. Because of this, we recommend adding the generated files to your `.gitignore` file:

```
*.service
SOURCES
SPECS
```

### Install your dependencies _first_

Specit assumes that you've _already installed your npm dependencies_ when it is run. This means that you don't need to worry about running `npm install` inside a clean RPM-building environment like _mock_.

The generated spec file instructs your RPM building tool to run [`npm rebuild`](https://docs.npmjs.com/cli/rebuild) as part of the build process. This ensures that any native modules are rebuilt for your target environment, even if they were originally installed on a different platform.

A typical specit build looks like this:

```bash
npm install
npm test
specit
# build the RPM (using rpmbuild, mock etc.)
```

### Local installation

To avoid the need to install specit globally, we recommend installing it _locally_ and creating an [npm script](https://docs.npmjs.com/misc/scripts) in your `package.json` file:

```
npm install --save-dev specit
```

```json
{
  "scripts": {
    "spec": "specit"
  }
}
```

You can then run `npm run spec` to generate your spec file in an environment where specit isn't installed globally (like your CI server.)

### Custom Spec Template

If you want to use a different specfile template to create your package, you can specify it your `package.json`:
```json
{
  "spec": {
    "specTemplate": "templates/myspec.mustache"
  }
}
```

### Build Architecture

By default Specit will build `noarch` packages meaning the final package should be installable on every CPU Architecture your system runs on. 
Specit's default template will also skip instruct rpmbuild to skip binary stripping during build.
If your Nodejs application has binary modules you may want to disable this behavior through your `package.json`.

```json
{
  "spec": {
    "noarch": false  
  }
}
```

### Pruning dependencies

To minimise the final RPM size, your development dependencies (dependencies added with the --save-dev flag) are automatically [pruned](https://docs.npmjs.com/cli/prune) so that they're not shipped with your production code.

If for some reason you need to package your dev dependencies with your production code you can explicity tell specit not to prune by adding the following to your `package.json`:

```json
{
  "spec": {
    "prune": false  
  }
}
```

### `npm start` script

The systemd service file that Specit generates uses the `npm start` script to start your application. Make sure that you've defined this script in your `package.json` file.

```json
{
  "scripts": {
    "start": "node server.js"
  }
}
```

### Node versions

By default, the spec file that specit generates _isn't_ tied to a particular Node version. It simply requires the `nodejs` package. It's up to you to make the package available when you install the RPM using `yum`.

We **strongly recommend** that you use the [Nodesource binary distributions](https://github.com/nodesource/distributions) to install a modern version of Node.js for both your RPM building environment and your target server. Follow the setup instructions for [Enterprise Linux](https://github.com/nodesource/distributions#rpm) and then run `yum install nodejs`.

If you're using multiple node repositories or a repository with multiple versions of node, you can specify an RPM version requirement with the `nodeVersion` property in your `package.json` file:

```json
{
  "spec": {
    "nodeVersion": "< 5.0.0"
  }
}
```
The `nodeVersion` property must conform to the [RPM version syntax](http://www.rpm.org/max-rpm/s1-rpm-depend-manual-dependencies.html#S3-RPM-DEPEND-VERSION-REQUIREMENTS)

### Directory Structure

Specit creates the following directories for your application:

|Directory|Purpose|
|---------|-------|
|`/usr/lib/:projectName`|This is where your application is stored|
|`/var/log/:projectName`|This is created for any log files that your application needs to write to|

## Configuration

Specit is configured using the `spec` property inside your existing `package.json` file.

### Dependencies

To add a dependency to the generated spec file, list the package dependencies in the `requires` array:

```json
{
  "spec": {
    "requires": [
      "vim",
      "screen"
    ]
  }
}
```

If you have any build dependencies (such as `python` for `node-gyp`), instead of having them available outside the build environment you can instead add them to the `buildRequires` array:

```json
{
  "spec": {
    "buildRequires": [
      "python"
    ]
  }
}
```

### Build scripts

If you need to perform any actions after installing your package (such as moving files on the target server) you can specify these inline using the `post` property:

```json
{
  "spec": {
    "build": [
      "gulp clean",
      "gulp build"
    ]
  }
}
```

### Executables

If you have scripts that need to be executable when they're installed on your target server, add them to the `executable` array. You can list both files and entire directories:

```json
{
  "spec": {
    "executable": [
      "./other-scripts/my-script.js",
      "./scripts"
    ]
  }
}
```

### Post Install Actions

If you need to perform any actions after installing your package (such as moving files on the target server) you can specify these inline using the `post` property:

```json
{
  "spec": {
    "post": [
      "mv /usr/lib/my-cool-api/rc.local /etc/rc.local"
    ]
  }
}
```

### Environment variable

If you need to specify environment variables during startup (NODE_ENV for example) you can specify these inline using the spec.environment property:

```json
{
  "spec": {
    "environment": {
      "NODE_ENV": "production",
      "NODE_INSTANCE": "%i"
    }
  }
}
```

### Release Number

By default specit will set the RPM release number to 1, if you want to override this you can do so by using the `--release` flag:

```sh
specit --release=7
```

### Custom Name

By default specit will set the name from `package.json`, if you want to override this you can do so by using the `--name` flag:

```sh
specit --name=my-cool-api
```

This is useful if you are using private NPM packages which start with an `@`.

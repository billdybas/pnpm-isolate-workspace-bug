# pnpm-isolate-workspace Bug

https://github.com/Madvinking/pnpm-isolate-workspace

Made for the following issues:
- https://github.com/Madvinking/pnpm-isolate-workspace/issues/7
- https://github.com/Madvinking/pnpm-isolate-workspace/issues/8

## Get Started

This project assumes you have [pnpm](https://pnpm.io/) installed.

```sh
pnpm bootstrap
pnpm isolate @isolate/package-1
```

## Bug

In `packages/package-1/_isolated_/workspaces`, we'll note that `package-3` is not copied over despite being a `devDependency` of `package-2`.

The `pnpm-lock.yaml` correctly marks `package-3` as a devDependency with a relative link, but there are no files present in the `workspaces` directory for this package.

```yaml
workspaces/packages/package-2:
    specifiers:
      '@isolate/package-3': workspace:*
      tslib: ^2.3.1
    dependencies:
      tslib: 2.3.1
    devDependencies:
      '@isolate/package-3': link:../package-3
```

As a result, if you try to `pnpm install --frozen-lockfile` inside of the `_isolated_` folder, you get an error from pnpm.

```
Lockfile is up-to-date, resolution step is skipped
 ERROR  Cannot install with "frozen-lockfile" because pnpm-lock.yaml is not up-to-date with workspaces/packages/package-2/package.json
```

Similarly, if you look at the `package.json` of `packages/package-1/_isolated_/workspaces/package-2`, it doesn't list `package-3` here.

```json
{
  "private": true,
  "name": "@isolate/package-2",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "package-3"
  },
  "dependencies": {
    "tslib": "^2.3.1"
  },
  "devDependencies": {}
}
```

I also notice the `pnpm-workspace.yaml` does _not_ contain `package-3`.

```yaml
packages:
  - workspaces/packages/package-2
```

## Fix

### Option 1

If you add `package-3` as a devDependency and copy over its source into the `workspaces` directory, running `pnpm install --frozen-lockfile` works as expected.

```shell
cp -r packages/package-3 packages/package-1/_isolated_/workspaces/packages
```

```shell
echo "`jq '.devDependencies."@isolate/package-3"="workspace:*"' packages/package-1/_isolated_/workspaces/packages/package-2/package.json`" > packages/package-1/_isolated_/workspaces/packages/package-2/package.json
```

```json
{
  "private": true,
  "name": "@isolate/package-2",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "package-3"
  },
  "dependencies": {
    "tslib": "^2.3.1"
  },
  "devDependencies": {
    "@isolate/package-3": "workspace:*"
  }
}
```

Now, `pnpm install --frozen-lockfile`.

### Option 2

Change the dependency on `package-3` in `package-2` to be a dependency instead of a dev dependency. When isolating, it will be correctly copied over into `_isolated_`.

```json
{
  "private": true,
  "name": "@isolate/package-2",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "package-3"
  },
  "dependencies": {
    "@isolate/package-3": "workspace:*",
    "tslib": "^2.3.1"
  }
}

```

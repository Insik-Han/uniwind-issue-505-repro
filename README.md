# uniwind-issue-505-repro

Minimal reproduction for [uni-stack/uniwind#505](https://github.com/uni-stack/uniwind/issues/505):

> Metro `Unexpectedly escaped traversal` invariant in `nativeResolver` during Expo eager bundle.

## Summary

On its first invocation, `nativeResolver` in `uniwind/dist/metro/index.cjs`
calls `resolver(context, "uniwind/components", platform)` to compute
`cachedInternalBasePath`. When the Expo eager bundle
(`expo export:embed --eager --dev false`) runs inside a pnpm workspace,
Metro hands `nativeResolver` a `context.originModulePath` that points at
the workspace root (e.g. `/abs/path/to/monorepo/.`). Metro's
`redirectModulePath` then calls `getPackageForModule(originModulePath)`,
which in turn invokes `TreeFS#hierarchicalLookup` relative to the current
TreeFS root (`apps/my-app/`). Because the origin path lives **above** that
tree root, TreeFS throws:

```
Invariant Violation: Unexpectedly escaped traversal
    at TreeFS.#lookupByNormalPath (metro-file-map/src/lib/TreeFS.js:536)
    ...
    at nativeResolver (uniwind/dist/metro/index.cjs:81)
```

`expo start` / dev mode does **not** reproduce the failure because Metro's
dev resolver path is lazier and bypasses `redirectModulePath`'s
package-scoped walk-up.

## Repo layout

```
uniwind-issue-505-repro/
├── apps/my-app/        Expo app (SDK 55 / RN 0.81 / uniwind 1.6.2)
├── .npmrc              node-linker=hoisted
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
└── README.md
```

As in the upstream issue, `.npmrc` pins `node-linker=hoisted` so the
node_modules layout is hoisted. The invariant also reproduces **without**
`.npmrc` (pnpm default = isolated linker), confirming that the root cause
is not the hoisting strategy but the shape of running an eager bundle from
`apps/my-app` inside a pnpm workspace.

## Versions

Expo SDK 55.

| Package / Tool          | Installed  | Declared in `package.json` |
|-------------------------|------------|----------------------------|
| pnpm                    | 10.33.0    | —                          |
| expo                    | 55.0.17    | ~55.0.17                   |
| @expo/cli               | 55.0.26    | (transitive)               |
| react                   | 19.2.0     | 19.2.0                     |
| react-native            | 0.83.6     | 0.83.6                     |
| react-native-reanimated | 4.2.1      | ^4.2.1                     |
| nativewind              | 4.2.3      | ^4.2.3                     |
| tailwindcss             | 4.2.4      | ^4.2.4                     |
| uniwind                 | 1.6.2      | ^1.6.2                     |
| expo-status-bar         | 55.0.5     | ~55.0.5                    |

## How to reproduce

```sh
# 1. install deps
pnpm install

# 2. run the eager bundle from apps/my-app (NOT from the workspace root)
cd apps/my-app
pnpm expo export:embed \
  --eager \
  --platform ios \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/main.jsbundle \
  --assets-dest /tmp/assets \
  --reset-cache
```

### Expected (after fix)

The bundle finishes successfully
(`Writing bundle output to: /tmp/main.jsbundle`).

### Actual

The bundle fails immediately after "Starting Metro Bundler" with:

```
Unexpectedly escaped traversal
Invariant Violation: Unexpectedly escaped traversal
    at TreeFS.#lookupByNormalPath (.../metro-file-map/src/lib/TreeFS.js:536:28)
    at TreeFS.hierarchicalLookup (.../metro-file-map/src/lib/TreeFS.js:552:51)
    at DependencyGraph._getClosestPackage (.../metro/src/node-haste/DependencyGraph.js:180:37)
    ...
    at redirectModulePath (.../metro-resolver/src/PackageResolve.js:41:29)
    at resolve (.../metro-resolver/src/resolve.js:98:65)
    ...
    at nativeResolver (.../uniwind/dist/metro/index.cjs:81:34)
    at firstResolver (.../uniwind/dist/metro/index.cjs:165:26)
```

## Suggested fix (from the upstream issue)

`nativeResolver` only needs its own installation root in that branch.
Resolving it through Metro's resolver is unnecessary and
context-dependent. Node's resolver works and is context-independent:

```diff
   const resolution = resolver(context, moduleName, platform);
   if (cachedInternalBasePath === null) {
-    const componentsResolution = resolver(context, "uniwind/components", platform);
-    cachedInternalBasePath = componentsResolution.type === "sourceFile"
-      ? node_path.join(node_path.dirname(componentsResolution.filePath), "../..")
-      : "";
+    try {
+      cachedInternalBasePath = node_path.dirname(require.resolve("uniwind/package.json"));
+    } catch (_e) {
+      cachedInternalBasePath = "";
+    }
   }
```

`webResolver` (around lines 110-112 in the same file) has the same pattern
with `"../../../.."`, which resolves to the same uniwind root. It should
be patched similarly for symmetry (Web reproduction not verified here).

Verified via `pnpm patch uniwind@1.6.2`: with the diff above applied, the
eager bundle completes successfully (2894 modules in ~13s) and
`eas build -p ios -e development --local` passes end-to-end.

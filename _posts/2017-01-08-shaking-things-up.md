---
layout: post
title: Shaking Things Up with Shake and Shakers in a Simple Haskell Project
---

{{ page.title }}
================

*8 January 2017*

[shake][shake] is a Haskell library for writing build systems to replace `make`, supporting rich and expressive project building functionality. We use `shake` in our Haskell projects to provide various simple and complex rules:

+ Formatting, linting, preprocessing, compiling source
+ Running tests
+ Installing binaries
+ Publishing libraries
+ Migrating databases
+ Building containers
+ Deploying applications
+ Running utilities

To avoid boilerplate and promote reusability among our Haskell projects, we have developed [shakers][shakers] as a Haskell library around `shake` to capture defaults and common build patterns. This post walks through setting up `shake` in a simple Haskell project with the help of `shakers`. The complete code for the project can be found at [shakers-example][shakers-example].

Shakers
-------

Before walking through a simple Haskell project setup, this section introduces `shakers` and its motivations. In building our Haskell projects, we have followed a common set of conventions that we have collected in `shakers` to help standardize our builds: `shakers` helps bootstrap our Haskell projects and provides them with similar, consistent build environments.

```haskell
-- | File containing project's shake source.
shakeFile :: FilePath
shakeFile = "Shakefile.hs"

-- | Location of supporting shake files.
buildDir :: FilePath
buildDir = ".build"

-- | Wrap shakeArgs with default options - main shake entry point.
shakeMain :: Rules () -> IO ()
shakeMain act = do
  version <- getHashedShakeVersion [shakeFile]
  let options = shakeOptions {shakeFiles=buildDir, shakeVersion=version, shakeThreads=0}
  shakeArgs options $ do
    -- see below
```

Our Haskell projects have a `Shakefile.hs` in their root directory which is used to version builds - when the contents of the file changes, complete rebuilds will be forced. Supporting shake files are stored in `.build` and include shake's database, **fake** files, **meta** files, and mirrored project directories for building containers. Projects use `shakeMain` in their files as their main entry point for running shake.

```haskell
-- | Directory where fake files are kept.
fakeDir :: FilePath
fakeDir = buildDir </> "fake"

-- | Fake file path builder.
fakeFile :: FilePath -> FilePath
fakeFile = (fakeDir </>)

-- | Use a fake file to keep track of file-free actions.
fake :: [FilePattern] -> String -> ([FilePath] -> Action ()) -> Rules ()
fake pats target act =
  fakeFile target %> \out -> do
    files <- getDirectoryFiles "." pats
    act files
    writeFile' out mempty

-- | Wraps a fake target with a phony target.
fake' :: [FilePattern] -> String -> ([FilePath] -> Action ()) -> Rules ()
fake' pats target act = do
  fake pats target act

  phony target $
    need [fakeFile target]
```

Fake targets are actions that depend on files but do not generate files and do not need to be rebuilt unless their dependent files have changed. A `lint` target could be an example fake target - no lint file is generated and lint does not need to be rerun unless the files being linted have changed. `fake'` is a convenience adding user-accessible phony targets with fake targets.

```haskell
-- | Directory where meta files are kept.
metaDir :: FilePath
metaDir = buildDir </> "meta"

-- | Meta file path builder.
metaFile :: FilePath -> FilePath
metaFile = (metaDir </>)

-- | Use a meta file to keep track of generated content actions.
meta :: FilePath -> Action String -> Rules ()
meta target act =
  metaFile target %> \out -> do
    alwaysRerun
    content <- act
    writeFileChanged out content
```

Meta targets are actions that depend on generated content and do not need to be rebuilt unless the generated content changes. A compiler version target could be an example meta target, as explained in the shake [manual][shake-manual].

```haskell
-- | Chomp excess space.
rstrip :: String -> String
rstrip = reverse . dropWhile isSpace . reverse

-- | Typed command args with return.
cmdArgs :: String -> [String] -> Action String
cmdArgs c as = rstrip . fromStdout <$> cmd c as

-- | Typed command args with no return.
cmdArgs_ :: String -> [String] -> Action ()
cmdArgs_ c as = unit $ cmd c as

-- | Run command in a directory with return.
cmdArgsDir :: FilePath -> String -> [String] -> Action String
cmdArgsDir d c as = rstrip . fromStdout <$> cmd (Cwd d) c as

-- | Run command in a directory with no return.
cmdArgsDir_ :: FilePath -> String -> [String] -> Action ()
cmdArgsDir_ d c as = unit $ cmd (Cwd d) c as

-- | Stack command with return.
stack :: [String] -> Action String
stack = cmdArgs "stack"

-- | Stack command with no return.
stack_ :: [String] -> Action ()
stack_ = cmdArgs_ "stack"

-- | Stack exec command with return.
stackExec :: String -> [String] -> Action String
stackExec c as = stack $ "exec" : c : "--" : as

-- | Stack exec command with no return.
stackExec_ :: String -> [String] -> Action ()
stackExec_ c as = stack_ $ "exec" : c : "--" : as

-- | Sylish command.
stylish_ :: [String] -> Action ()
stylish_ = cmdArgs_ "stylish-haskell"

-- | Lint command.
lint_ :: [String] -> Action ()
lint_ = cmdArgs_ "hlint"

-- | Git command.
git :: [String] -> Action String
git = cmdArgs "git"

-- | m4 command.
m4 :: [String] -> Action String
m4 = cmdArgs "m4"

-- | Rsync command.
rsync_ :: [String] -> Action ()
rsync_ = cmdArgs_ "rsync"

-- | SSH command with return.
ssh :: String -> [String] -> Action String
ssh h as = cmdArgs "ssh" $ h : as

-- | SSH command with no return.
ssh_ :: String -> [String] -> Action ()
ssh_ h as = cmdArgs_ "ssh" $ h : as

-- | SSH command in a remote directory with return.
sshDir :: String -> FilePath -> [String] -> Action String
sshDir h d as = ssh h $ "cd" : d : "&&" : as

-- | SSH command in a remote directory with no return.
sshDir_ :: String -> FilePath -> [String] -> Action ()
sshDir_ h d as = ssh_ h $ "cd" : d : "&&" : as

-- | Git tag version.
gitVersion :: Action String
gitVersion = git ["describe", "--tags", "--abbrev=0"]

```

A number of common commands are captured and defined as a convenience for use in shake actions.

```haskell
-- | Preprocess files with a template and m4.
preprocess :: FilePattern -> FilePath -> Action [(String, String)] -> Rules ()
preprocess target file kvs =
  target %> \out -> do
    need [file]
    let f k v = "-D" <> k <> "=" <> v
    kvs' <- kvs
    content <- m4 $ file : (uncurry f <$> kvs')
    writeFileChanged out content
```

Preprocess targets are actions that depend on a template and generate files from the template with `m4` and key value pairs of defines. Generating a cabal file and its version from a template could be an example preprocess target.

```haskell
-- | Haskell source rules.
hsRules :: Rules ()
hsRules = do
  let pats = ["//*.hs"]

  -- | format - run stylish over Haskell source.
  fake' pats "format" $ \files -> do
    need [".stylish-haskell.yaml"]
    stylish_ $ ["-c", ".stylish-haskell.yaml", "-i"] <> files

  -- | lint - run hlint over Haskell source.
  fake' pats "lint" $ \files ->
    lint_ files
```

TODO write about haskell source rules.

```haskell
-- | Default rules.
shakeRules :: Rules ()
shakeRules = do
  -- | clear - remove fake and meta files so targets can rebuild.
  phony "clear" $
    forM_ [fakeDir, metaDir] $
      flip removeFilesAfter ["//*"]

  -- | clean - remove supporting shake files and stack work.
  phony "clean" $ do
    stack_ ["clean"]
    removeFilesAfter buildDir ["//*"]
```

TODO write about shake rules.

```haskell
-- | Stack rules.
stackRules :: [FilePattern] -> Rules ()
stackRules pats = do
  -- | build - compile source.
  fake' pats "build" $ const $
    stack_ ["build", "--fast"]

  -- | build-error - compile source with -Werror.
  fake' pats "build-error" $ const $
    stack_ ["build", "--fast", "--ghc-options=-Werror"]

  -- | build-tests - compile source and tests.
  fake' pats "build-tests" $ const $
    stack_ ["build", "--fast", "--test", "--no-run-tests"]

  -- | build-tests-error - compile source and tests with -Werror.
  fake' pats "build-tests-error" $ const $
    stack_ ["build", "--fast", "--test", "--no-run-tests", "--ghc-options=-Werror"]

  -- | install - copy binaries.
  fake' pats "install" $ const $
    stack_ ["build", "--fast", "--copy-bins"]

  -- | tests - compile and run tests.
  phony "tests" $
    stack_ ["build", "--fast", "--test"]

  -- | tests-error - compile run tests with -Werror.
  phony "tests-error" $
    stack_ ["build", "--fast", "--test", "--ghc-options=-Werror"]

  -- | ghci - start REPL with source.
  phony "ghci" $
    stack_ ["ghci", "--fast"]

  -- | ghci-tests - start REPL with source and tests.
  phony "ghci-tests" $
    stack_ ["ghci", "--fast", "--test"]
```

TODO write about stack rules.

```haskell
-- | Cabal rules.
cabalRules :: FilePath -> Rules ()
cabalRules file = do
  -- | gitVersion - meta file of current git tag version.
  meta "gitVersion" gitVersion

  -- | cabal - generate cabal file from template.
  preprocess file (file <.> "m4") $ do
    need [metaFile "gitVersion"]
    version <- gitVersion
    return [("VERSION", version)]

  -- | publish - publish to hackage.
  phony "publish" $ do
    need [file]
    stack_ ["sdist"]
    stack_ ["upload", ".", "--no-signature"]
```

TODO write about cabal rules.

```haskell
-- | Wrap shakeArgs with default options - main shake entry point.
shakeMain :: Rules () -> IO ()
shakeMain act = do
  version <- getHashedShakeVersion [shakeFile]
  let options = shakeOptions {shakeFiles=buildDir, shakeVersion=version, shakeThreads=0}
  shakeArgs options $ do
    shakeRules
    hsRules
    act
```

TODO write about shakeMain again.

Example Project
---------------

TODO write about example project.

```haskell
#!/usr/bin/env stack
{- stack
    runghc
    --package shakers
 -}

import Development.Shakers

-- | Main entry point.
main :: IO ()
main = shakeMain $ do
  let pats =
        [ "stack.yaml"
        , "Shakefile.hs"
        , "main//*.hs"
        ]

  -- | Cabal rules.
  cabalRules "shakers-example.cabal"

  -- | Stack rules.
  stackRules pats

  -- | sanity - ensure everything's ok.
  fake' pats "sanity" $ const $
    need [ fakeFile "build-error", fakeFile "lint" ]

  -- | Default things to run.
  want [ fakeFile "sanity", fakeFile "format" ]
```

TODO write more about example project.


Interested in working on infrastructure in Haskell? Contact me!

[shake]:           http://hackage.haskell.org/package/shake
[shakers]:         http://hackage.haskell.org/package/shakers
[shakers-example]: https://github.com/mfine/shakers-example
[shake-manual]:    http://shakebuild.com/manual#dependencies-on-extra-information

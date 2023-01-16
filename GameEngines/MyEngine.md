## Arch goals

- data drive not OOP, so read the Data-Oriented Design book

## Features to add

- logging with time stamps
- ini settings to configure the game
- flame graph and other profiling tools
- result types like in rust (use lib)


## Workflow

- conan package for the engine, consumed by the game
- semver versioning (verlite or something else)
- github actions to build
- github actions to create release packages (deploys)
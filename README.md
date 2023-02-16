<img src="./logo.png" width="500">

---

This style guide captures what Kurtosis views as good code. It was created descriptively, from years of writing Kurtosis code; every section resulted from encounters "in the wild"; none of it is aspirational "we should do this but nobody really does".

Other language-specific style guides:
- [Golang](./guides/go-style-guide.md)
- [Typescript](./guides/typescript-style-guide.md)
- [Bash](./guides/bash-style-guide.md)

Principles
----------
The core principle of Kurtosis code is "flexible minimalism", and [you should read this to understand what that means and why](./flexible-minimalism.md).

Several ideas fall out of this:

- Code should be easy to understand 
- Explicit code is always better than implicit code
- Verbose code that's easy to understand is better than "clever", compact code that's difficult to understand
- Less complex = simpler, and simpler is always better

Good tests to measure if you're writing Kurtosis-quality code:

- Are you building only against the requirements we have?
- Could someone understand your code by PR alone, without type or parameter hints from an IDE?
- Imagine your code is next changed by a tired, stressed Kurtosisian at midnight 6 months from now (the "Future Midnight Kurtosian"). Is your code clear and defensive enough to keep them safe?

Structual Guidance
------------------
### Remove dead code (dead code is dangerous code)
Sometimes refactors will make code obsolete. If you're going to use the code within the next week, comment it out. Otherwise, delete it.

Rationale:
* Keeping dead code up-to-date on the off chance that we might need it someday is just-in-case engineering
* If someone did actually end up using the code again, the dead code has likely not been maintained and will be buggy
* Maintaining dead code takes up time that could be used for maintaining live code

### Make things as private as possible
This is important to:
- Reduce the complexity of the public-facing API (i.e., makes the API more self-guiding which aligns with our principle above of forcing people into good choices)
- Reduce dependency tangling (i.e. reduce the chance that someone depends on something public that they shouldn't)

**NOTE:** this is particularly important for functions. The preference order for functions should be:

1. Private static functions (easy to test, but private so they don't end up in the global namespace)
1. Private instance functions (attached to a class)
1. Public instance functions
1. Public static functions

We call these public static functions "utility" functions, as they're just chunks of logic floating off in space. They're particularly dangerous for several reasons:

1. They're not discoverable: if somebody needs their functionality, they have to magically know that the function exists
1. They promote more utility functions, which reduces the natural readability of the code ("Rather than thinking about where my new function should go, I'll just toss it in with all the other utility functions...")
1. They promote unnecessary dependencies ("Ah, I see that the `kubernetes` package has a `TrimStringLength` function that I could use; I'll just pull that entire dependency in...")

### Replace "magic constants" with constants named for their purpose (NOT their value)
"Magic constants" are constants that come out of nowhere and make code harder to read, e.g. the `true` in `dockerManager.GetContainersByLabels(labels, true)`. If the magic constant is left in place, a reader won't know what argument that `true` is filling in or if it's the correct value. Many things can be magic constants, not just booleans - e.g. integers, strings, map literals, etc. 

To fix, give the constant a name that reflects its purpose:

```
const (
    shouldGetStoppedContainersWhenGettingContainersByLabel = true
)

dockerManager.GetContainersByLabels(labels, shouldGetStoppedContainersWhenGettingContainersByLabel)
```

Why do we emphasize the "purpose"? When engineers first start naming their constants, the natural tendency is to name it after the value:

```
const (
    trueValue = true
)

dockerManager.GetContainersByLabels(labels, trueValue)
```

This has three problems: 

1. It doesn't make the code where the constant is used any clearer to read
1. If you change this value then you need to refactor the constant name (which means changing the code everywhere the constant is used)
1. It tempts future engineers to use the constant for completely different purposes (see the "Over-DRYing" section elsewhere in this guide)

If we instead name the constant after its purpose (`shouldGetStoppedContainersWhenGettingContainersByLabel`) then the code becomes clearer and we're free to change the value as needed without the risk that somebody used the value for something completely different.

EXCEPTION: Tests are the only place where we don't follow this rigorously, as tests often require declaring many, many constants.

### Default to "eject early, eject often"
Compare these two versions of the same code:

```go
// Version 1
client, err := CreateNewClient()
if err == nil {
    logrus.Info("Created the client successfully")
    result, err := client.DoThing()
    if err == nil {
        logrus.Info("Did the thing successfully")
    } else {
        return stacktrace.NewError("DoThing failed")
    }
} else {
    return stacktrace.NewError("An error occurred creating the client")
}
```
```go
// Version 2
client, err := CreateNewClient()
if err != nil {
    return stacktrace.NewError("An error occurred creating the client")
}
logrus.Info("Created the client successfully")
result, err := client.DoThing()
if err != nil {
    return stacktrace.NewError("DoThing failed")
}
logrus.Info("Did the thing successfully")
```

In the first version (which is the common way most people think - "success first, then error"), the branching quickly gets out of control. In the second, we do the error-checking first and eject if errors are found, then are guaranteed that the subsequent code below the error-checking is error-free.

This is known as reducing the [cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) - i.e. the number of distinct paths that the program can go down - and it results in easier-to-understand code! It is also basically the same as [the "indent error flow" section of Google's Go style guide](https://github.com/golang/go/wiki/CodeReviewComments#indent-error-flow).

### Pick up your toys
See the [dedicated page about the rules of resource ownership](./guides/picking-up-your-toys.md)

### Avoid too much nested complexity (aka "TMNC") by using local variables
Nested complexity is when a single line is doing many things. E.g.:

```go
something.SomeFunc(
    fmt.Sprintf(
        "%v %v",
        getSomeVariableValue(),
        strings.Join(someSlice, " "),
    ),
    NewThingBuilder().WithParam1(
        fmt.Sprintf(
            "%v",
            someUintValue,
        ),
    ).WithParam2(
        getOtherVariableValue()
    ).Build()
)
```

Even though the code is formatted well, there's just _so_ much going on that it's hard to follow. Local variables are the easy solution for TMNC:

```go
repeatedSomethingIdValuesStr := strings.Join(someSlice, " ")
somethingId := fmt.Sprintf(
    "%v %v",
    getSomeVariableValue(),
    repeatedSomethingIdValuesStr,
)

thingParam1 := fmt.Sprintf(
    "%v",
    someUintValue,
)
thingParam2 := getOtherVariableValue()
thing := NewThingBuilder().WithParam1(
    thingParam1,
).WithParam2(
    thingParam2,
).Build()

something.SomeFunc(
    somethingId,
    thing,
)
```

Notice how it's easier to tell what's going on, _and_ the code has become more self-documenting because the interstitial values are named.

### Make your objects immutable whenever possible
This reduces the number of states the object can be in, which will make thinking about the code & debugging significantly easier. This also reduces [cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity).

### Avoid doing logic in your constructors whenever possible
Generally, constructors should be "dumb" - just take in parameters and pass them to the object being created. They shouldn't do complex things like creating database connections or generating IDs, because objects with constructors that do things are difficult to test - e.g. if your constructor autogenerates an ID, how do you test an object with a specific ID value? Or if your constructor creates a database connection, how in testing do you pass in a mock database connection?

This goes hand-in-hand with the design principle ["favor composition over inheritance"](https://en.wikipedia.org/wiki/Composition_over_inheritance).

### When propagating errors, log the parameters of the function that failed
This makes debugging much easier, especially if the error is difficult to reproduce. Example in Go:

```golang
if err := someFunc(value1, value2); err != nil {
    // Obviously, use human-readable descriptions in the real world
    return stacktrace.Propagate(
        err, 
        "An error occurred calling someFunc with param1 '%v' and param2 '%v'",
        value1,
        value2,
    )
}
```

### When logging parameters, make sure that the parameter is easy to read
This means default to surrounding parameters with `''` quotes so that the parameters are clear, even if `value1` or `value2` have a space in them.

For parameters expected to have a newline, it's often easiest to put it on its own line. E.g. in Go:

```golang
if err := doSomething(); err != nil {
    logrus.Errorf("An error occurred doing something:\n%v", err)
}
```

### Use loggers whenever possible
Loggers are great: you can control where they route output to, you can have them output in a structured machine-readable format, and you get fine-grained control over which log levels are printed.

### Name your lambdas when feasible
This would be normal Go code to write:

```go
type dumpContainerResult struct {
    logs string
    containerInspectJsonStr string
    err error
}

func DumpContainers() {
    // ...get containers...

    allDumpResults := make(chan dumpContainerResult)
    for containerId := range allContainerIds {
        go func(containerId string) {
            logs, err := getContainerLogs(containerId)
            if err != nil {
                allDumpResults <- stacktrace.Propagate(err, "An error occurred getting logs for container '%v'", containerId)
                return
            }

            inspectJsonStr, err := inspectContainer(containerId)
            if err != nil {
                allDumpResults <- stacktrace.Propagate(err, "An error occurred inspecting container '%v'", containerId)
                return
            }

            allDumpResults <- dumpContainerResult{
                logs: logs,
                containerInspectJsonStr: inspectJsonStr,
                err: nil,
            }
        }(containerId)
    }

    // ....process results...
}
```

However, there are quite a few levels of nested complexity here (see the "too much nested complexity" section). Breaking the lambda out into a named helper function cleans this up significantly:

```go
func DumpContainers() {
    // ...get containers...

    allDumpResults := make(chan dumpContainerResult)
    for containerId := range allContainerIds {
        go dumpContainer(containerId, allDumpResults)
    }

    // ....process results...
}

func dumpContainer(containerId string, allDumpResults chan dumpContainerResult) {
    logs, err := getContainerLogs(containerId)
    if err != nil {
        allDumpResults <- stacktrace.Propagate(err, "An error occurred getting logs for container '%v'", containerId)
        return
    }

    inspectJsonStr, err := inspectContainer(containerId)
    if err != nil {
        allDumpResults <- stacktrace.Propagate(err, "An error occurred inspecting container '%v'", containerId)
        return
    }

    allDumpResults <- dumpContainerResult{
        logs: logs,
        containerInspectJsonStr: inspectJsonStr,
        err: nil,
    }
}
```

Note that this isn't always feasible - e.g. in Go, naming tiny `defer` functions would make the code harder to read, not easier.

### Don't add preemptive debug statements, but do leave your debug statements in
It's very difficult to anticipate in advance where you'll need debug statements. You could go crazy and add debug statements everywhere, but this clutters the code and oftentimes isn't even necessary due to our philosophy around logging parameters when returning errors. Adding preemptive debug statements therefore violates our principle of minimalism.

However, you will inevitably need to add debug/trace logs when debugging something in the wild. You should add your statements to find your problems, and then leave them in the codebase. The fact that you had to add them indicates that this piece of code is problematic enough to require debug/trace logging, which increases the likelihood that someone else will need to look at this in the future.

### Use human-readable semantic information in log/error messages, rather than internal symbols
For example, rather than a log message like:
```
Failed to deserialize PortSpecInfo '%+v' on call to validatePortSpec
```
do one like:
```
Failed to deserialize port spec information '%+v' when validating port spec
```
This has two benefits: 1) you don't need to hunt down all log messages when you rename a struct or function 2) the error messages are human-readable, which is good because users can see them.

### Don't "over-DRY" your code and create connections between components when none are needed
Whenever two things look the same, it's tempting to follow DRY and immediately refactor them into a centralized place. This is often a good idea, but isn't always.

Suppose we have the following function on the Docker client:

```go
func (client *DockerClient) GetContainers(shouldGetStoppedContainers bool) ([]*Container, error) {
    // ...
}
```

and we have several places that use magic constants:

```go
// inside our "logs_collector_functions.go"
logsCollectorContainers, err := dockerClient.GetContainers(false)

// inside our "engine_functions.go"
engineContainers, err := dockerClient.GetContainers(false)


// inside our "module_functions.go"
moduleContainers, err := dockerClient.GetContainers(true)

// inside our "user_service_functions.go"
userServiceContainers, err := dockerClient.GetContainers(true)
```

It's tempting to look at the `true` and `false` values, think "I should DRY these!", and attempt to extract them into two constants - one with `true` and one with `false`. However, the error is that these are all _semantically different values_, even if they share the same value. The clue for realizing this is, what would you even call the constant? `shouldGetStoppedContainersTrueValue`? Then you're naming a constant by its value rather than its semantic value, which violates our principle of "name constants off their purpose".

The solution is four constants, because there are four semantically unique values:

```go
// inside our "logs_collector_functions.go"
const (
    shouldGetStoppedLogsCollectorContainers = false
)
logsCollectorContainers, err := dockerClient.GetContainers(shouldGetStoppedLogsCollectorContainers)

// inside our "engine_functions.go"
const (
    shouldGetStoppedEngineContainers = false
)
engineContainers, err := dockerClient.GetContainers(shouldGetStoppedEngineContainers)


// inside our "module_functions.go"
const (
    shouldGetStoppedModuleContainers = true
)
moduleContainers, err := dockerClient.GetContainers(shouldGetStoppedModuleContainers)

// inside our "user_service_functions.go"
const (
    shouldGetStoppedUserServiceContainers = true
)
userServiceContainers, err := dockerClient.GetContainers(shouldGetStoppedUserServiceContainers)
```

For another example of the same idea in practice, [see here](https://gordonc.bearblog.dev/dry-most-over-rated-programming-principle/#:~:text=DRY%20is%20misused%20to%20eliminate%20coincidental%20repetition). 

**Conclusion: don't DRY things that have the same value, but different purposes**

### Do not use forbidden knowledge (obey the principles of abstraction)
A caller of a component should only need to rely on the contract that the component declares, and must know or care how the component is implemented. Relying on internal implementation details of the component is using "forbidden knowledge" about the component. 

Likewise, a component should not know the particulars of how it is being used. Doing otherwise relies on "forbidden knowledge" about who is calling the component. An example here would be modifying a component to expose information about its internal state because it would be very useful to a caller of the component. Instead, you should think, "What is this component trying to do, and does its purpose need to be changed?"

The point of the "no forbidden knowledge" rule is to provide clear API boundaries between two components, so that it's easy to swap out or rewrite a component. This promotes codebase flexibility.

### Keep the Kurtosis SDKs in various languages as identical as possible!
Kurtosis maintains SDKs in multiple languages to provide the best possible experience to a developer. However, each SDK incurs maintenance surface area: updating upon changes, fixing bugs, upgrading dependencies, etc. 

To minimize our developer burden, we strive to keep the SDKs **as identical as possible** - even down to the exact language of log messages, variable & function naming, etc. This level of stringency might seem excessive, but the goal is to allow the Future Midnight Kurtosian to make changes to all the SDKs as brainlessly as possible (which includes copying log messages from SDK to SDK). Put another way, our code should still be flexible even for the Future Midnight Kurtosian.

### When doing bulk operations, strongly prefer vertical slicing over horizontal slicing
Imagine that you're writing a `StartServices` function that starts multiple services in bulk:

```go
// Returns:
//  - set of successful IDs
//  - set of failed IDs, with their error
func StartServices(servicConfigs map[ServiceID]*ServiceConfig) (map[ServiceID]bool, map[ServiceID]error) {
    // TODO
}
```

Each service has three phases it needs to proceed through - IP allocation, files artifact expansion, and container starting. Your function will therefore be attempting to check boxes in a graph like this:

```
|--------------------|--------------|--------------|--------------|--------|
|        Phase       | Service ID 1 | Service ID 2 | Service ID 3 | ...... |
|--------------------|--------------|--------------|--------------|--------|
|    IP Allocation   |              |              |              |        |
|--------------------|--------------|--------------|--------------|--------|
| Artifact Expansion |              |              |              |        |
|--------------------|--------------|--------------|--------------|--------|
| Container Starting |              |              |              |        |
|--------------------|--------------|--------------|--------------|--------|
```

There are two ways to tackle the problem:

1. Slicing it **horizontally**, by phase, so that all service IDs are pushed through IP allocation, then the successful ones are pushed through artifact expansion, then _those_ successful ones are pushed through container starting
1. Slicing it **vertically**, by service ID, so that each service ID is pushed all the way through before the next one is attempted

(Note that this slicing is _independent_ of parallelism; both approaches can be single-threaded or parallelized!)

We always prefer the vertical slicing, because it's much easier to reason about, especially [once defer-undo'd resource cleanup comes into play][picking-up-your-toys]. To see why, let's look at what each of these would look like.

First, horizontal slicing:

```go
func StartServices(servicConfigs map[ServiceID]*ServiceConfig) (map[ServiceID]bool, map[ServiceID]error) {
    failedServiceIds := map[ServiceID]error{}

    // IP allocation
    serviceIpAddresses := map[ServiceID]*net.IP{}
    pendingIpAddressDeallocs := map[ServiceID]bool{}
    defer func() {
        for serviceId := range pendingIpAddressDeallocs {
            // Should actually check the 'found' value of the map, but omitting for example brevity
            deallocateIpAddress(serviceIpAddresses[serviceId])
        }
    }
    for serviceId := range serviceConfigs {
        ipAddress, err := allocateIpAddress()
        if err != nil {
            failedServiceIds[serviceId] = stacktrace.Propagate("An error occurred allocating an IP address for service '%v'", serviceId)
            continue
        }
        serviceIpAddresses[serviceId] = ipAddress
        pendingIpAddressDeallocs[serviceId] = true
    }

    // Files artifact expansion
    serviceFilesArtifacts := map[ServiceID]map[FilesArtifactID]bool{}
    pendingFilesArtifactDeletions := map[ServiceID]bool{}
    defer func() {
        for serviceId := range pendingFilesArtifactDeletions {
            // Should actually check the 'found' value of the map, but omitting for example brevity
            deleteFilesArtifacts(serviceFilesArtifacts[serviceId])
        }
    }
    for serviceId, serviceConfig := range serviceConfigs {
        filesArtifactIds, err := expandFilesArtifacts(serviceConfig.FilesArtifactsConfig)
        if err != nil {
            failedServiceIds[serviceId] = stacktrace.Propagate("An error occurred expanding files artifacts for service '%v'", serviceId)
            continue
        }
        serviceFilesArtifacts[serviceId] = filesArtifactIds
        pendingFilesArtifactDeletions[serviceId] = true
    }

    // Starting containers
    serviceContainers := map[ServiceID]ContainerID{}
    pendingContainerDeletions := map[ServiceID]bool{}
    defer func() {
        for serviceId := range pendingContainerDeletions {
            // Should actually check the 'found' value of the map, but omitting for example brevity
            deleteContainer(serviceContainers[serviceId])
        }
    }
    for serviceId, serviceConfig := range serviceConfigs {
        containerId, err := startContainer(serviceConfig.ContainerConfig)
        if err != nil {
            failedServiceIds[serviceId] = stacktrace.Propagate("An error occurred starting container for service '%v'", serviceId)
            continue
        }
        serviceContainers[serviceId] = containerId
        pendingContainerDeletions[serviceId] = true
    }

    successfulServiceIds := map[ServiceID]bool{}
    for serviceId := range serviceContainers {
        successfulServiceIds[serviceId] = true
    }

    for serviceId := range successful {
        delete(ipAddressesToDeallocate, serviceId)
        delete(pendingFilesArtifactDeletions, serviceId)
        delete(pendingContainerDeletions, serviceId)
    }
    return successfulServiceIds, failedServiceIds
}
```

We call this approach the "funnelling" approach, where service IDs funnel from one phase to the next. It _works_, but it's a big mess and difficult to reason about - especially with all the defer-undo'd resource deletion.

Now, vertical slicing:

```go
func StartServices(servicConfigs map[ServiceID]*ServiceConfig) (map[ServiceID]bool, map[ServiceID]error) {
    successfulServiceIds := map[ServiceID]bool{}
    failedServiceIds := map[ServiceID]error{}
    for serviceId, config := range serviceConfigs {
        if err := startService(serviceId, config); err != nil {
            failedServiceIds[serviceId] = stacktrace.Propagate(err, "An error occurred starting service '%v'", serviceId)
            continue
        }
        successfulServiceIds[serviceId] = true
    }
    return successfulServiceIds, failedServiceIds
}

func startService(serviceId ServiceID, config *ServiceConfig) error {
    // IP allocation
    ipAddress, err := allocateIpAddress()
    if err != nil {
        return stacktrace.Propagate("An error occurred allocating an IP address for service '%v'", serviceId)
    }
    shouldDeallocateIpAddress := true
    defer func() {
        if shouldDeallocateIpAddress {
            deallocateIpAddress(ipAddress)
        }
    }

    // Files artifact expansion
    filesArtifactIds, err := expandFilesArtifacts(config.FilesArtifactsConfig)
    if err != nil {
        return stacktrace.Propagate("An error occurred expanding files artifacts for service '%v'", serviceId)
    }
    shouldDeleteFilesArtifacts := true
    defer func() {
        if shouldDeleteFilesArtifacts {
            deleteFilesArtifacts(ipAddress)
        }
    }

    containerId, err := startContainer(config.ContainerConfig)
    if err != nil {
        return stacktrace.Propagate("An error occurred starting container for service '%v'", serviceId)
    }
    shouldDeleteContainer := true
    defer func() {
        if shouldDeleteContainer {
            deleteContainer(ipAddress)
        }
    }

    shouldDeallocateIpAddress := false
    shouldDeleteFilesArtifacts := false
    shouldDeleteContainer := false
    return nil
}
```

Much cleaner, and it lends itself nicely to parallelization - simply task a bunch of `startService` calls against a threadpool.

**NOTE:** For a while we weren't sure which was better, so you'll likely see old examples of horizontal slicing in the codebase. These should be converted to vertical slicing.

### Use absolute file/directory paths by default
This will help ensure your code works when being called from any directory.

### When stubbing a function out, have it fail noisily
If you're adding a new function that you don't want to implement just yet, it's tempting to have it be a no-op (e.g. `return nil` in Go). However, if you forget that you did this for any reason (maybe the weekend happened or maybe you went on vacation), you'll spend time hunting down a confusing no-op just to feel silly when you find your stub. This is not hypothetical; we've already had it happen before.

Instead, have your function return a big loud error indicating that you still need to implement it - once your code is hit, you'll be reminded immediately that you need to implement it without any debugging. This has extra benefit of following the principle "force people into good choices; don't rely on memory".

Visual Guidance
---------------
### Follow language-specific naming/capitalization conventions
E.g.:
- Go should use `camelCase` with `acronymsLikeIDCapitalized` and `camelCasedEnums`
- Typescript and Javascript should use `UpperCamelCase` classes, leaving `acronymsLikeIdUncapitalized`, and `UPPER_SNAKE_CASE` enums
- Python and Starlark should use `lower_snake_case` for variables and functions and `ProperCase` for classes

### Start your function names with verbs
Functions do things, and starting every function with a verb a) forces you to think about what exactly that function should be doing and b) serves as a source of self-documentation to users.

### When writing a class, all public functions come first, followed by private functions
This makes the public API very easy to see, rather than being forced to pick through all the functions to see which are public and which are private

### Name your objects according to their narrow purpose
Objects should be named according to their purpose. To check if your object has a good name, run these tests:

1) Do the object's functions make sense with the object name?
2) If I saw my object name in a PR without any other context, what would I expect it to do? Does its actual purpose match that? 

An example from a PR:

```golang
type LogsDatabase interface {
	GetPrivateHttpPortSpec() (*port_spec.PortSpec, error)
	GetContainerArgs(
		containerName string,
		containerLabels map[string]string,
		volumeName string,
		networkId string,
	) (*docker_manager.CreateAndStartContainerArgs, error)
	GetConfigContent() (string, error)
}
```

Let's apply the tests:

1) Do the function match go with the object name? Yes, the functions more or less go with the `LogsDatabase` name although the functions are quite low-level (more applicable to "a general HTTP container with a config file" than a logs database specifically)
2) If I saw `LogsDatabase` in the code, would my expected functions match the actual functions? No; `LogsDatabase` would lend itself to functions like `GetLogs`, `SearchLogs`, and `DeleteLogs`

Using these tests, we can see that this object is actually for getting container configuration for a logs database. `LogsDatabaseContainerConfigProvider` would be a better name.

### When naming file/directory path variables, use the {dir,file}{name,path} convention, prefixed with {abs,rel} as necessary
This helps make clear what the variable is expected to contain. For example:
* A variable containing the path to a config file would be called `configFilepath` in Go and `config_filepath` in Bash
* A variable containing the path to a config directory would be called `configDirpath` in Go and `config_dirpath` in Bash
* A variable containing just the name of the config file would be called `configFilename` in Go
* A variable containing just the name of a config directory would be called `configDirname` in Go
* A variable containing a path to a config file, relative to the repo's root, would be called `relConfigDirpath` (and should contain a comment pointing to what it's relative to)
* When relative paths are involved, the config file path can be shown to be absolute with `absConfigFilepath` (by default, paths are assumed to be absolute so the `abs` can be omitted when no relative paths are involved)

### Give boolean variables, and functions that return booleans, names that are questions
This makes the type visually obvious when reading in a PR. An example based off a real PR:

```
clusterSet, err := clusterSettingStore.GetClusterSet()
```

This makes it seem like we're getting a set of clusters, but this code is actually returning a boolean indicating whether a cluster has been set.
Now compare the updated code:

```
isClusterSet, err := clusterSettingStore.IsClusterSet()
```

This code is more explicit, and therefore more visually obvious.

### Use one of our two styles for functions with many arguments

When a function has many arguments, you can either leave it on a single line or break it up to one argument per line like so:

```
// Option 1
myFunctionCall(arg1, arg2, arg3, arg4, arg5, arg6)

// Option 2
myFunctionCall(
    arg1, 
    arg2, 
    arg3, 
    arg4, 
    arg5, 
    arg6
)

// BAD: not obvious what the args are; closing parenthesis isn't on newline
myFunctionCall(arg1, 
    arg2, arg3, arg4, 
    arg5, arg6)
```

When chaining methods, use whichever option your language permits:

```
// Java style, which permit lines to start with '.'
functionCall1(arg1_1, arg1_2, arg1_3)
    .functionCall2(arg2)
    .functionCall3(arg3)

// Python/Go style, which don't permit lines to start with '.'
functionCall1(
    arg1_1,
    arg1_2,
    arg1_3
).functionCall2(
    arg2
).functionCall3(arg3)
```

### Cuddle your braces
Rationale: it drastically reduces the number of lines of code, meaning you can see more content.

```
// DO THIS
if (someCondition) {
    ....
} else if (otherCondition) {
    ....
} else {
    ....
}

// DON'T DO THIS
if (someCondition)
{
    .....
}
else if (otherCondition)
{
    .....
}
else
    .....
}
```



<!--------------------------------- ONLY LINKS BELOW HERE ------------------------------------------>
[picking-up-your-toys]: ./picking-up-your-toys.md


<!----------------------------------- NOTHING BELOW HERE ------------------------------------------->

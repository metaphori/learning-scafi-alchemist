# Notes

This document includes notes about the use of Alchemist, Protelis, Scafi, and whatever was used in order to implement these experiments.

Versions

```kotlin
    implementation("it.unibo.alchemist:alchemist:9.0.0")
    implementation("it.unibo.alchemist:alchemist-incarnation-scafi:9.0.0")
    implementation("org.scala-lang:scala-library:2.12.2")
    implementation("it.unibo.apice.scafiteam:scafi-core_2.12:0.3.2")
    implementation("org.protelis:protelis-lang:13.0.2")
```

## Alchemist

### Summary

- **Molecule**: name of a data item
- **Concentration**: value associated to a particular molecule
- **Node**: a container of molecules/reactions, living inside an environment
- **Environment**: the Alchemist abstration for the space.
    - It is a container for nodes, and it is able to tell:
  a) Where the nodes are in the space - i.e. their position
  b) How distant are two nodes
  c) Optionally, it may provide support for moving nodes
- **Linking rule**: a function of the current status of the environment that associates to each node a
  neighborhood
    - **Neighborhood**: an entity composed by a node (centre) + a set of nodes (neighbors)
- **Reaction**: any event that can change (through an **action**) the state of the environment
    - Consists of 0+ **conditions**, 1+ **actions**, and a **time distribution**
    - Conditions, time distribution, static rate, and rate equation affect the **frequency** of the reaction
- Alchemist implements an optimised version (NRM) of Gillespie's Stochastic Simulation Algorithm (SSA)

So

- The **system state** depends on the configuration of molecules floating in it
- The **system evolution** depends on the kinds of chemical reactions applicable over time

Another key concept is the **dependency graph**

- Actions are outputs
- Conditions are inputs

### Where Alchemist finds stuff

- It uses **default packages**; e.g., for actions, it is **`it.unibo.alchemist.model.implementations.actions`** (if you use another package, you must use FQCNs in the YAML)

## Simulations 


##### Graphs

Alchemist internally uses **[jgrapht](https://jgrapht.org/)**.

##### Launching

NOTE: this currently does not work (USE GRADLE TASKS)

```bash
./gradlew fatJar # or shadowJar

java -Xmx2524m -cp "build/libs/experiment19-ac-slcs-rv-redist.jar" \
  it.unibo.alchemist.Alchemist \
  -b -var random \
  -y src/main/yaml/monitoringTraces.yml -e data/20191004-test \
  -t 100 -p 1 -v &> exec.txt &
```

##### Maps

```yaml
variables:
  range: &range
    type: GeometricVariable
    parameters: [1000, 10, 1000, 9] # default, min, max, num samples
    # e.g., increasingly bigger radiuses to simulate the processing of more data on cloud
moveFrequency: &moveFrequency
    formula: 1/60

ReproduceGPSTrace: &ReproduceGPSTrace
  type: ReproduceGPSTrace
  parameters: ["vcmuser.gpx", false, AlignToTime, 1365922800, false, false] 
            # (1) path to file with GPS traces, 
            # (2)bool saying if traces have to be distributed cyclically,
            # (3,...) class that implement the strategy to normalize the time + corresponding args

environment:
  type: OSMEnvironment # Open Street Map Environment
  parameters: ["vcm.pbf", false] 
  # Args: file, onstreets (if true, nodes are places on nearest street)
  # Possibly, a third, boolean arg can be provided: onlyOnStreets---
  # if true, the nodes which are too far from a street will be simply discarded. 
  # If false, they will be placed anyway, in the original position.                                                                    

network-model:
  type: ConnectWithinDistance
  parameters: [*range]
  
pools:
  - pool: &moveWithTraces
    - time-distribution: 0.1 # 0.1 means 1/10, i.e., once every 10 units of simulated time
      type: Event
      actions:
      - *ReproduceGPSTrace
      
displacements:
- in:
    type: FromGPSTrace
    parameters: [1497, "vcmuser.gpx", false, AlignToTime, 1365922800, false, false]
    # numNodes, path to GPS traces file (GPX-GPs eXchange format), cycle, normalizer+args
  programs:
  - *moveWithTraces
  contents:
  - molecule: test
    concentration: true
```

- `FromGPSTrace` displacement: distributes nodes in the first positions of a `GPSTrace`
- [GPX (GPS Exchange Format)](https://en.wikipedia.org/wiki/GPS_Exchange_Format), is an XML schema designed as a common GPS data format for SW apps. 
  It can be used to describe waypoints, tracks, and routes.
- The `environment` is what affects the choice of the kind of coordinates and the unit of measures
- For `OSMEnvironment`, the range in `ConnectWithinDistance` is in metres
    - For a Wi-Fi connection the range should be < 100; so, for a realistic setting, you should use access points.
    There's a displacement class, `CloseToAlreadyDisplaced` (args: num nodes, variance i.e. what distance from nodes you tolerate) or `CloseToGPSTrace`, that distributes nodes on the basis of the density of nodes that are already placed.
- On **geometric variables**: they are used to "sample in a logarithmic space"; so, e.g., if you want 4 samples in interval [0,10000] 
  you'll get the following sampling values [10, 10000, 100, 1000]; these values are considered in batch mode, otherwise the default value of the var is used.

Normalizers for `ReproduceGPSTrace`

- `AlignToFirstTrace`: **Aligns all traces at the start time of the first trace**. E.g., if you have two traces, the first trace start with time = 2 and second point with time = 5, the second trace start with time = 4 and second point with time = 6, the result will be: - first trace start with time = 0 and second point with time = 3 - second trace start with time = 2 and second point with time = 4`
- `AlignToSimulationTime`: **Aligns all traces at the initial simulation time**. E.g., if you have two traces, the first trace start with time = 2 and second point with time = 5, the second trace start with time = 4 and second point with time = 6, the result will be: - first trace start with time = 0 and second point with time = 3 - second trace start with time = 0 and second point with time = 2
- `AlignToTime`: **Alings the traces with the given time in seconds from Epoch**. E.g., all points before such time will be discarded. All points after the provided time will be shifted back. Summarizing, the time that is provided represents in the real world the time zero of the simulation.

In order to study "evacuation scenarios" and the like, 
it is convenient to **(1) put the nodes on streets** and **(2) making them move following streets**.
This can be done by initially putting nodes on streets, via

```yaml
environment:
  type: OSMEnvironment # Open Street Map Environment
  parameters: ["vcm.pbf", true] 
  # Args: file, onstreets (if true, nodes are places on nearest street)
```

and then using `GPSTraceWalker` instead of `ReproduceGPSTrace`:

```yaml
ReproduceGPSTraceOnStreets: &ReproduceGPSTraceOnStreets
  type: GPSTraceWalker
  parameters: ["vcmuser.gpx", false, AlignToTime, 1365922800, false, false] 
            # (1) path to file with GPS traces, 
            # (2)bool saying if traces have to be distributed cyclically,
            # (3,...) class that implement the strategy to normalize the time + corresponding args
```

[As noted in the Alchemist doc on maps](https://alchemistsimulator.github.io/wiki/usage/maps/),
there are many ways to mode nodes in an environment with a real-world map

1. **move ignoring map info**: from start position to end position
2. **move using map info**: move from start position to destination using intermediate position in order to follow streets and avoid obstacles (e.g., buildings)
3. **move by reproducing a GPS trace** (action `ReproduceGPSTrace`): the movement from a position to another is direct with a straight line
4. **move by interpolating the GPS trace with street data** (action `GPSTraceWalker`): the movement from a position to another use map information to define intermediate position in order to follow streets and avoid obstacle (like building)

##### Putting access points

```yaml
displacements:
  # PEOPLE
  - in:
      type: FromGPSTrace
      parameters: [*peopleCount, "vcmuser.gpx", false, AlignToTime, 1365922800, false, false]
    programs:
    - *moveWithTraces
    - *program
    contents:
    - molecule: range
      concentration: *range
    - molecule: accessPoint
      concentration: false
  # ACCESS POINTS
  - in:
      type: CloseToGPSTrace
      parameters: [*accessPointCount, 1e-5, 0, 60, 3600, "vcmuser.gpx", AlignToTime, 1365922800, false, false]
      # nodeCount, variance, fromTime, interval, toTime, gpsFilePath, normalizerClass+args
    programs:
    - *program
    contents:
    - molecule: accessPoint
      concentration: true
```

Note the **CloseToGPSTrace** displacement.

There is also a kind of network model based on APs

```yaml
network-model:
  type: ConnectToAccessPoint
  parameters: [*range, *accessPointId]
  # range, molecola 
```

- **ConnectToAccessPoint**: the AP is an intermediary of a communication between two nodes
- **ConnectViaAccessPoint**: two nodes can communicate *directly* through the AP
- TODO: add doc to Alchemist to explain `ConnectToAccessPoint` and `ConnectViaAccessPoint`.

Notice that in the scenario of Vienna you need *a lot* of APs to make things work.

- However, it is useful to show how increasing numbers of APs lead to better performance.
- Notice that, with few APs, you get *network partitions* but the algorithm may still work there.
- If you need a well-connected network, you may choose to connect APs using longer ranges, i.e., to have well-connected mesh of APs


Issue encountered:

```text
Exception in thread "main" java.lang.IllegalStateException: 
java.lang.IllegalArgumentException: Cannot create it.unibo.alchemist.loader.displacements.FromGPSTrace 
  with arguments [1947:Integer, vcmuser.gpx:String, false:Boolean, AlignToTime:String, 1365922800:Integer, false:Boolean, false:Boolean]
...
Caused by: java.lang.IllegalArgumentException: 1947traces required, 1503 traces available
```

##### Arbitrary variables (e.g., range over "a","b","c" in batch, with "a" by default)

**NOTE: CURRENTLY VERY FRAGILE**

This technique is very useful to configure multiple simulation scenarios:

```yaml
variables:
  currentCase: &currentCase
    type: ArbitraryVariable
    parameters: ["noAccessPoints", ["noAccessPoints","accessPointsMesh"]]
  connectionType: &connectionType
    formula:
      currentCase match {
        case "noAccessPoints" => "ConnectWithinDistance" // Note, these are strings
        case "accessPointsMesh" => "ConnectViaAccessPoint"
      }
    language: scala
    
network-model:
  type: *connectionType
  parameters: [*range, *accessPointId]
```

Issues encountered:

- `it.unibo.alchemist.loader.IllegalAlchemistYAMLException: it.unibo.alchemist.loader.variables.ArbitraryVariableis not a subclass of interface it.unibo.alchemist.loader.variables.DependentVariable`

##### Nodes at arbitrary locations

See `Point` displacement or `SpecificPositions`:

```yaml
- in:
    type: SpecificPositions #hospitals
    parameters: 
    # This format doesn't work [[48.2152226,16.3816613],[48.2366507,16.3838339]]
    # as the pairs are interpreted as ArrayLists
    - *program
  contents:
  - molecule: someMol
    concentration: someVal
```

##### Scripts and variables

If your script depends on variables, do this:

```yaml
variables:
  currentCase: 
    formula: 1
  noAPcase: 
    formula: 0
  apCase: 
    formula: 1
  accessPointCount: &accessPointCount
    formula:
      val s = currentCase;
      val case1 = noAPcase;
      val case2 = APcase;
      currentCase match {
        case `case1` => 0
        case `case2` => 500
      }
```

If you don't pre-declare the `val`s you may get errors as the following ones: `Caused by: javax.script.ScriptException: stable identifier required, but $iw.this.noAPcase found. 
in currentCase match { case noAPcase => 0 case APcase => 500 }`


## Protelis

#### Protelis in Alchemist

```yaml
pools:
- pool: &program
  - time-distribution: 1
    type: Event
    actions:
    - type: RunProtelisProgram
      parameters: ["crowd", *retentionTime]
  - program: send # DO NOT FORGET THIS TO ENABLE COMMUNICATION AMONG NODES
```

- `retentionTime`: how long the messages will be stored. Pass Double#NaN to mean that they should get eliminated upon node awake.

Sometimes you want to **access the environment from Protelis**

- **env** keyword gives access to Protelis' `ExecutionEnvironment`
    - `env.put('string', value)` to put a molecule on node
    - `env.get('string')` to get a molecule's concentration
    - **`env.putField(s, nbr(v))` to print a field** (useful, e.g., for debug) 
- **self** keyword gives access to Protelis'`ExecutionContext`, that is **`AlchemistExecutionContext`** for an Alchemist env
    - `self.getExecutionEnvironment()` also returns the current `ExecutionEnvironment`
    - `self.getDeviceUID().toString()`
    - `self.getCoordinates()`, in maps returns a tuple `t = [lat,long]`, so `let lat = t.get(0)`
- **`AlchemistExecutionContext.getEnvironmentAccess`** returns the Alchemist `Environment` (not to be confused with Protelis' `ExecutionEnvironment`)

#### On changing concentration of molecules

Molecules may be modified in a reaction.
A Protelis program may modify anything (i.e., any molecule) **in the current node**.


#### How to call Java code from Protelis?

**Protelis import mechanism is like Java's `import static`**: so you cannot import classes, you can only import objects or methods, `*` for all.

- Apparently, the following does not work
```
import System.out

println("Prova")
```
but this works
```
import java.lang.System.out

out.println("Hello wld");
```

- Btw, error reporting is misleading: it may tell you that *other* modules could not be imported

#### How to call Kotlin code from Protelis?

For instance, create a file with top-level Kotlin functions as follows

```kotlin
@file:JvmName("SimulationUtils")

fun f(){ ... }
```

and then import the corresponding Java class from Protelis:

```
import SimulationUtils.*

f()
```

#### Modules

```
module foo:bar:baz
```

for a file `foo/bar/baz.pt` from classpath' root.

Notice that your Protelis code directory, e.g., `src/main/protelis`, must be a source folder! So, e.g., in the Gradle build:

```
sourceSets.getByName("main") {
    resources {
        srcDirs("src/main/protelis")
    }
}
```


## Issues

- Running simulations in gradle 
    - Process is killed [due to memory issues](https://stackoverflow.com/questions/38967991/why-are-my-gradle-builds-dying-with-exit-code-137)
```text
> Process 'command '/usr/lib/jvm/java-11-oracle/bin/java'' finished with non-zero exit value 137
```
- Creating a fat or shadow JAR (using either shadowJar plugin for Gradle or the task in the Alchemist build),
  and then running Alchemist using that JAR in the classpath,
  results in the following error
```text
Exception in thread "main" java.lang.IllegalArgumentException: groovy is not an available language. Your environment supports the following languages:
 - ECMAScript, aka [js, application/javascript, application/ecmascript, text/javascript, text/ecmascript] (ECMA - 262 Edition 5.1 on Oracle Nashorn 11)
 - Scala, aka [scala, application/x-scala] (version 2.12.2 on Scala REPL 2.0)
```
- Gradients did not work because I forgot to **enable communication among nodes**
```yaml
pools:
  - pool: &crowd
    - time-distribution: 1
      type: Event
      actions:
        - type: RunProtelisProgram
          parameters: [crowd, *retentionTime, true]
    - time-distribution:
        type: SimpleNetworkArrivals
        parameters: [*propagationDelay, *packetSizeId, "", *bandwidthId, "", *accessPointId]
      program: send
```
- Debugging Protelis fields: see `NBRCall.evaluate`, which calls `context.buildField`


## Experience

- Pay attention to: network diameter
- Pay attention to: retention time 
<img src="https://raw.githubusercontent.com/harshadmanglani/Assets/master/Polaris.jpg">

An extremely light weight workflow orchestrator for Golang.

Inspired from https://github.com/flipkart-incubator/databuilderframework

### Getting Started
From the root of your Go module, run:
```
go get github.com/harshadmanglani/polaris@v1.0.0
```
### Usage

```go
// dataStore = DataStore{} - use your database by implementing the IDataStore interface
polaris.InitRegistry(dataStore)
polaris.RegisterWorkflow(workflowKey, workflow)

executor := polaris.Executor{
	Before: func(builder reflect.Type, delta []IData) {
		fmt.Printf("Builder %s is about to be run with new data %v\n", builder, delta)
	},
	After: func(builder reflect.Type, produced IData) {
		fmt.Printf("Builder %s produced %s\n", builder, produced)
	},
}

response, err := executor.Run(workflowKey, workflowId, dataDelta)
```

## Use cases
1. You have multi-step workflow executions where each step is dependent on data generated from previous steps.
2. Executions can span one request scope or multiple scopes.
3. Your systems works with reusable components that can be combined in different ways to generate different end-results.
4. Your workflows can pause, resume or even restart from the beginning.

## Limitations
1. Workflow versioning is tricky to implement:
   1. Unless you can afford a 100% downtime ensuring all active workflows move into a terminal state, deploying new code requires ensuring backward compatibility.
   2. What this means is - you'll need to a deploy a version of code that is backward compatible for older non terminal workflows while newer ones will execute on the new code.
   3. Once the older workflows have completed, a deployment to clean up stale code will be required.
2. The level of abstraction is lower in this framework compared to Cadence, Conductor:
   1. Workflows can be made fault oblivious if there is an external (reliable) service giving callbacks per workflow id.
   2. Instrumentation can be set up by adding your custom code to push events via listeners.

## Terminologies

* _**Data**_ - The basic container for information generated by an actor in the system.
* _**Builder**_ - An actor that consumes a bunch of data and produces another data. It has the following meta associated with it:
    * **Name** - Name of the builder
    * **Consumes** - A set of Data that the builder consumes
    * **Produces** - Data that the builder produces
    * **Optionals** - Data that the builder can optionally consume; one possible use case for this: if a builder wants to be re-run on demand with the same set of consumable Data already present, add an optional Data in the Builder and restart the workflow by passing an instance of the optional Data in the DataDelta
    * **Access** - Data that the builder will just access and has no effect on building the topologically sorted ExecutionGraph
* _**Workflow**_ - A specification and container for a topology of connected Builders that generate a final data. It has the following meta:
    * **Name** - Name of the workflow
    * **Target Data** - The name of the data being generated by this data flow
    * **Resolution Specs** - If multiple builders known to the system can produce the same data, then this can be used to put an override specifying which particular builder will generate a particular data in context of this data flow.
    * **Transients** - A set of names of Data that would be considered Transients for this case. (See later for a detailed explanation of transients)
* _**DependencyHierarchy**_ - A graph of connected and topologically sorted builders that are used by the execution engine to execute a flow. 
* _**DataSet**_ - A set of the Data provided by the client and generated internally by the different builders of a particular ExecutionGraph

## How does the framework perform at scale?
The framework itself has extremely low overhead. Since execution graphs are generated pre-runtime, all the orchestrator will do at runtime is use the graph and available data to run whichever builders can be run. Benchmarking pending.
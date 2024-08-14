# CFP-1189: Cilium CLI connectivity tests speedup

**SIG: SIG-USER**

**Begin Design Discussion:** 2024-01-19

**Status:** Released Cilium-cli 0.16

**Authors:** Viktor Kurchenko <viktor.kurchenko@isovalent.com>

## Summary

The CFP describes a new approach to run Cilium connectivity tests.
The idea is to group all the tests in small sets and run them in parallel.

## Motivation

Currently, connectivity tests might run in CI for over 1 hour (it depends
on many factors) and the test case count constantly increasing.
To make CI pipelines faster and cheaper we can consider the connectivity
tests parallelization approach.

## Goals

* Group connectivity tests into independent sets that can be run concurrently.
* Run each independent test set concurrently (all together or in batches).
* Collect test results and display them periodically.

## Proposal

### Walkthrough

1. Cilium CLI groups all connectivity tests into independent sets that
   do not interfere with each other. A separate component can be implemented
   to keep this responsibility (e.g.: `TestSetFactory`).
2. Each produced test set can be run concurrently. In some cases, it might be
   not acceptable to run many test sets concurrently (e.g.: due to limited
   resources in a cluster). CLI should provide an option
   (e.g.: `--test-batch-size`) that allows to group test sets into fixed-size
   batches and run batches synchronously.
3. Each test set should provision its namespace and all the required resources.
4. A separate component (e.g.: `TestMonitor`) can be implemented and run in a
   dedicated goroutine to collect each test set execution results and display
   them periodically.

### Concurrent output example

![Conn tests concurrent output](./images/conn-tests-concurrent-output.gif)

However, this example might not work properly on GitHub actions due to in-place
output updates. In pipelines, CLI can cache and display each test result
sequentially even with a predefined order.

+++
author = "Adrian"
title = "How to Memoize Results From useQueries Hook using React Query"
date = "2022-07-25"
description = "Memoize Results From useQueries Hook using React Query"
tags = [
    "rust",
    "react.js"
]
categories = [
    "technical"
]
series = ["Rust"]
aliases = ["rust"]
image = "bg.jpg"
+++



## How to Memoize Results From useQueries Hook using React Query

I started using React Query in my projects in late 2021, and I’ve been loving it so far. What’s not to like? Auto refetches, auto retries, client-side caching — I’m quite mad at myself for not trying this library sooner even though I’ve heard about it in the year prior. With that said, I’m writing this to share a very specific thing: how to memoize useQueries hook results to avoid triggering expensive recalculations downstream. Most of my projects at work involve data visualization, and often we need to process and visualize many data points. The size can vary greatly, from a single digit to low double digit data points (e.g., bar charts), 100s to 1000s (e.g. time series line charts, scatter plot, table), or even 10000+ (this is rare though — if any, typically only data analysts & scientists would find this useful). But even with only 100-1000 data points for each chart, often there can be multiple charts on a single page, making any sort of transformation a potential performance bottleneck. Ideally, these expensive transformations should be delegated to the backend, but sometimes there’s still some “last mile” logic such as reformatting the data to fit the required format of a charting library, or to compute layout and data-viz related attributes (e.g. size, color, etc) using D3, etc.
Needless to say, useMemo is the hero we need to avoid unnecessary computations, where the dependency array includes the queried data and potentially other state values(e.g. user interaction or filter-related states).
Most of the time, I only need React Query’s useQuery hook to run a single query to the backend/database. For example, let’s say we want to query some data to construct a bubble chart:
Here is a simple code sample

```js
// Assume we've configured the React Query client to have non-zero staleTime (e.g. 1000 * 60 * 60 * 24, or perhaps Infinity), otherwise it will always fetch new data and never uses the cached data. Reference: https://react-query.tanstack.com/guides/important-defaults
import React from "react";
import {useQuery} from "react-query";
// suppose this is an HTTP request to the backend/database, and
function fetchSomething() {
  returns a long list of data points
return Promise.resolve([
    { name: "data point 1", value: 1 },
    { name: "data point 2", value: 2 }
  ]);
}

// a hook to get your data points
function useDataPointsSingleGroup() {
  // react query hook
  const { data } = useQuery({
    queryKey: ["some-query"],  
    queryFn: fetchSomething
  });

  // some calculation against the queried data
  const decoratedData = React.useMemo(() => {
    if (!data) return [];
    // This can be whatever expensive operations you might need
    // (e.g. sort, map, groupBy, d3 scale or transformation, etc to construct a data visualization(s))
    return data.map((d) => ({
      ...d,
      attribute: "some attribute",
      color: "some color derived from some color scale",
      derivedMetric: 42,
      radius: 42, // for the bubble size
    }));
  }, [data]);
  return decoratedData;
}
```

Whenever React Query uses a cached result, the data object is the same. Therefore the decorated data calculation won’t be unnecessarily triggered since data is in the dependency array.
But what if we want to use useQueries to run multiple queries in parallel?

```js
import {useQuery} from "react-query";
// In this example the list is hardcoded. This could've also been a state or props that changes dynamically based on a user interaction or dashboard filters. 
const dataGroups = ["groupA", "groupB", "groupC"];
function useDataPointsMultiGroups() {
  const results = useQueries(
    dataGroups.map((dg) => {
      return {
        //note that we have to make the key unique per group
        queryKey: ["some-query", dg], 
        queryFn: fetchSomething
      };
    })
  );
  // `Results` is an array, and it's always a new array object. What should be in useMemo's dependency?
  ...
}
```

This is trickier because the object returned by useQueries is always a new array. The data object within each object of the array, however, is still the same object when the cache is retrieved.

```js
// the returned `results` array:
// this array is always a new object
[
 {  
    status: 'success',
    isLoading: false,
    data: [...] // but this `data` array is the same object if cache is used
  },
  {  
    status: 'success',
    isLoading: false,
    data: [...]
  },
  ...
]
```

It seems that we need to unpack this before feeding it to useMemo . First I had to define a utility hook called useArrayMemo :

```js
function useArrayMemo(array) {
  // this holds reference to previous value 
  const ref = React.useRef();
  // check if each element of the old and new array match
  const areArraysConsideredTheSame =
    ref.current && array.length === ref.current.length
      ? array.every((element, i) => {
        return element === ref.current[i];
      })
    //initially there's no old array defined/stored, so set to false
    : false; 
  React.useEffect(() => {
    //only update prev results if array is not deemed the same
    if (!areArraysConsideredTheSame) {
      ref.current = array;
    }
  }, [areArraysConsideredTheSame, array]);
  return areArraysConsideredTheSame ? ref.current : array;
}
```

We now can use it like this:

```js
const dataGroups = ["groupA", "groupB", "groupC"];
function useDataPointsMultiGroups() {
  const results = useQueries(
    dataGroups.map((dg) => {
      return {
        queryKey: ["some-query", dg],
        queryFn: fetchSomething
      };
    })
  );
  const dataSets = useArrayMemo(
    results.map((result) => result.data)
  );
  const decoratedData = React.useMemo(() => {
  //perhaps similar transformation as in the first example, but note that the input is an array of array
    
    ...
  }, [dataSets]);
  
  return decoratedData
}

```

Side note: That hook might also be useful for other things outside React Query whenever we want to check reference equality of elements of an array and, if so, use the cached array object. I just haven’t found a need to use it elsewhere.

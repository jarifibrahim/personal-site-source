---
title: "Golden Files — Why you should use them"
date: 2018-03-19T12:03:06+05:30
tags: ["Golang","Golden-Files", "Software-Testing"]
slug: "Golden Files — Why you should use them"
---
<style>
.caption {
    font-size: 0.9em;
    margin: 0px 50px;
    text-align: center;
    margin-bottom: 20px;
}
</style>

<div>
![my image](/golden-files.png)
<div class="caption">A Golden file from the fabric8 project — https://github.com/fabric8-services/fabric8-wit/blob/master/controller/test-files/label/update/ok.label.golden.json</div>
</div>
Testing responses from an API is often straightforward and monotonous. You set a few headers, make a request and assert the received response. The problem starts when your API sends a huge amount of data in the response. You validate **each attribute** of the response against the expected value. This often leads to bloated code that mostly consists of a lot of assert statements. A much better approach to testing such responses is using the **Golden Files**.

>A Golden File is the absolute source of truth. You validate your response against the Golden File. In nutshell, the Golden file contains the response you expect from your API.

## Talk is Cheap, Here’s the Code
```go
var update = flag.Bool("update", false, "update .golden.json files")
func TestRespose(t *testing.T) {
    response := DoSomething()    // Make API calls
    goldenFile := "ok.golden.json"
    // If update is set, write to the golden file
    if *update {
        ioutil.WriteFile(goldenFile, response, 0644)
    }
    expected, err := ioutil.ReadFile(goldenFile)
    if err != nil {
        // Handle error
    }
    if !bytes.Equal(goldenFile, response) {
        // Test Failure
    }
}
```

We’ve essentially reduced the test size by more than 70%. Instead of having
multiple assert statements throughout the test, we now have a single
`bytes.Equal` that validates the entire response data.

The update flag would allow us to create/update the golden files when the API response changes.

The workflow for running the tests now becomes —

- Run `go test` and make sure the test fails.
- Ensure the response from your API is accurate.
- Run `go test -update` to update the existing golden file (or create new ones).
- Re-run `go test` to ensure the test does not fail now.

With this simple piece of code, your tests are now much cleaner and easier to maintain.

>Be smart, use Golden Files.

## The Problem — ID and Timestamp
One of the major issues with using the Golden files is dealing with the ID and Timestamp. When the response from an API contains the unique ID representing the resource and its Timestamp, the golden file pattern would fail since these two entities aren’t static. The Timestamp and ID would never be same for any two entities (in normal circumstances). For instance, if you create a new resource, the timestamp would be the current time while the timestamp in the golden file would be the time when the golden file was created. This makes it very difficult to use the golden file for data with ID and Timestamp.

## But, wait — There’s a way to deal with ID and Timestamps
An elegant way of dealing with ID and Timestamps is to replace all the

- ID with `0001`, `0002`, `0003`, `....` , `000N` and,
- `Timestamps` with `0001–01–01T00:00:00Z` or `Mon, 01 Jan 0001 00:00:00 GMT`

Let’s see the code —
```go
// Compares the given actualObj with the goldenFile
func CompareWithGolden(t *testing.T, goldenFile string, actualObj interface{}) {
    expected, _ := ioutil.ReadFile(goldenFile)
    expectedStr := string(expected)
    actualStr := string(actualObj)

    // Replace ID
    expectedStr, _ = replaceIDs(expectedStr)
    actualStr, _ = replaceIDs(actualStr)
    // Replace Timestamp
    expectedStr, _ = replaceTimes(expectedStr)
    actualStr, _ = replaceTimes(actualStr)
    if expectedStr != actualStr {
        // Test Failed
    }
}
// findIDs returns an array of unique IDs that have been found in  // the given string
func findIDs(str string) ([]id, error){
    pattern := "^\d{4}$"
    idRegexp, err := regexp.Compile(pattern)
    uniqIDs := map[id]struct{}{}
    var res []id
    for _, idStr := range idRegexp.FindAllString(str, -1) {
        ID, _ := id.FromString(idStr)
        _, alreadyInMap := uniqIDs[ID]
        if !alreadyInMap {
            uniqIDs[ID] = struct{}{}
           // append to array
           res = append(res, ID)
        }
    }
    return res, nil
}
// replaceIDs finds all IDs in the given string and replaces them  // with 0001, 0002, 0003, ...., 000N
func replaceIDs(str string) (string, error) {
    replacementPattern := "%04d"
    ids, err := findIDs(str)
    newStr := str
    for idx, id := range ids {
        newStr = strings.Replace(
            newStr,
            id.String(),
            fmt.Sprintf(replacementPattern, idx+1),
            -1)
    }
    return newStr, nil
}
func replaceTimes(str string) (string, error) {
    // Works similar to replaceIDs method
}
```

The `IDs` and `Timestamps` generated by the above code snippet would be same for
the `actual` and the `expected` data. Thus, your tests would no longer fail
because of `IDs` or `Timestamps`.

The Above code snippet was taken from — https://github.com/fabric8-services/fabric8-wit/blob/master/controller/golden_files_test.go
(Thanks to the ever-awesome [Konrad Kleine](https://github.com/kwk) for writing this piece of code)

You can find more examples of Golden Files on — https://github.com/fabric8-services/fabric8-wit/tree/master/controller/test-files


---
More information

- http://vincent.demeester.fr/posts/2017-04-22-golang-testing-golden-file/
- https://medium.com/@povilasve/go-advanced-tips-tricks-a872503ac859

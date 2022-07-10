[![Go Report Card](https://goreportcard.com/badge/josephburnett/jd)](https://goreportcard.com/report/josephburnett/jd)

# JSON diff and patch

`jd` is a commandline utility and Go library for diffing and patching JSON and YAML values. It supports a native `jd` format (similar to unified format) as well as JSON Merge Patch ([RFC 7386](https://datatracker.ietf.org/doc/html/rfc7386)) and a subset of JSON Patch ([RFC 6902](https://datatracker.ietf.org/doc/html/rfc6902)). Try it out at [play.jd-tool.io](http://play.jd-tool.io/.)

![jd logo](assets/logo_small.png)

## Installation

To get the `jd` commandline utility:
* run `brew install jd`, or
* run `go install github.com/josephburnett/jd@latest`, or
* visit https://github.com/josephburnett/jd/releases/latest and download the pre-built binary for your architecture/os, or
* run in a Docker image `jd(){ docker run --rm -i -v $PWD:$PWD -w $PWD josephburnett/jd "$@"; }`.

To use the `jd` web UI:
* visit http://play.jd-tool.io/, or
* run `jd -port 8080` and visit http://localhost:8080.

## Command line usage

```
Usage: jd [OPTION]... FILE1 [FILE2]
Diff and patch JSON files.

Prints the diff of FILE1 and FILE2 to STDOUT.
When FILE2 is omitted the second input is read from STDIN.
When patching (-p) FILE1 is a diff.

Options:
  -color     Print color diff.
  -p         Apply patch FILE1 to FILE2 or STDIN.
  -o=FILE3   Write to FILE3 instead of STDOUT.
  -set       Treat arrays as sets.
  -mset      Treat arrays as multisets (bags).
  -setkeys   Keys to identify set objects
  -yaml      Read and write YAML instead of JSON.
  -port=N    Serve web UI on port N
  -f=FORMAT  Produce diff in FORMAT "jd" (default), "patch" (RFC 6902) or
             "merge" (RFC 7386)
  -t=FORMATS Translate FILE1 between FORMATS. Supported formats are "jd",
             "patch" (RFC 6902), "merge" (RFC 7386), "json" and "yaml".
             FORMATS are provided as a pair separated by "2". E.g.
             "yaml2json" or "jd2patch".

Examples:
  jd a.json b.json
  cat b.json | jd a.json
  jd -o patch a.json b.json; jd patch a.json
  jd -set a.json b.json
  jd -f patch a.json b.json
  jd -f merge a.json b.json
```

## Library usage

Note: import only release commits (`v1.Y.Z`) because `master` can be unstable.

```Go
import (
	"fmt"
	jd "github.com/josephburnett/jd/lib"
)

func ExampleJsonNode_Diff() {
	a, _ := jd.ReadJsonString(`{"foo":"bar"}`)
	b, _ := jd.ReadJsonString(`{"foo":"baz"}`)
	fmt.Print(a.Diff(b).Render())
	// Output:
	// @ ["foo"]
	// - "bar"
	// + "baz"
}

func ExampleJsonNode_Patch() {
	a, _ := jd.ReadJsonString(`["foo"]`)
	diff, _ := jd.ReadDiffString(`` +
		`@ [1]` + "\n" +
		`+ "bar"` + "\n")
	b, _ := a.Patch(diff)
	fmt.Print(b.Json())
	// Output:
	// ["foo","bar"]
}
```

## Diff language

![Railroad diagram of EBNF](assets/ebnf.png)

- A diff is zero or more sections
- Sections start with a `@` header and the path to a node
- A path is a JSON list of zero or more elements accessing collections
- A JSON number element (e.g. `0`) accesses an array
- A JSON string element (e.g. `"foo"`) accesses an object
- An empty JSON object element (`{}`) accesses an array as a set or multiset
- After the path is one or more removals or additions, removals first
- Removals start with `-` and then the JSON value to be removed
- Additions start with `+` and then the JSON value to added

### EBNF

```EBNF
Diff ::= ( '@' '[' ( 'JSON String' | 'JSON Number' | 'Empty JSON Object' )* ']' '\n' ( ( '-' 'JSON Value' '\n' )+ | '+' 'JSON Value' '\n' ) ( '+' 'JSON Value' '\n' )* )*
```

### Examples

```DIFF
@ ["a"]
- 1
+ 2
```

```DIFF
@ [2]
+ {"foo":"bar"}
```

```DIFF
@ ["Movies",67,"Title"]
- "Dr. Strangelove"
+ "Dr. Evil Love"
@ ["Movies",67,"Actors","Dr. Strangelove"]
- "Peter Sellers"
+ "Mike Myers"
@ ["Movies",102]
+ {"Title":"Austin Powers","Actors":{"Austin Powers":"Mike Myers"}}
```

```DIFF
@ ["Movies",67,"Tags",{}]
- "Romance"
+ "Action"
+ "Comedy"
```

## Cookbook

### Use git diff to produce a structural diff:
```
git difftool -yx jd @ -- foo.json
@ ["foo"]
- "bar"
+ "baz"
```

### See what changes in a Kubernetes Deployment:
```
kubectl get deployment example -oyaml > a.yaml
kubectl edit deployment example
# change cpu resource from 100m to 200m
kubectl get deployment example -oyaml | jd -yaml a.yaml
```
output:
```diff
@ ["metadata","annotations","deployment.kubernetes.io/revision"]
- "2"
+ "3"
@ ["metadata","generation"]
- 2
+ 3
@ ["metadata","resourceVersion"]
- "4661"
+ "5179"
@ ["spec","template","spec","containers",0,"resources","requests","cpu"]
- "100m"
+ "200m"
@ ["status","conditions",1,"lastUpdateTime"]
- "2021-12-23T09:40:39Z"
+ "2021-12-23T09:41:49Z"
@ ["status","conditions",1,"message"]
- "ReplicaSet \"nginx-deployment-787d795676\" has successfully progressed."
+ "ReplicaSet \"nginx-deployment-795c7f5bb\" has successfully progressed."
@ ["status","observedGeneration"]
- 2
+ 3
```
apply these change to another deployment:
```
# edit file "patch" to contain only the hunk updating cpu request
kubectl patch deployment example2 --type json --patch "$(jd -t jd2patch ~/patch)"
```

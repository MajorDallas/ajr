# ajr
Another JQ REPL using FZF

Inspired by the tool [Up](https://github.com/akavel/up) and the [jq-zsh-plugin](https://github.com/reegnz/jq-zsh-plugin),
this is a simple attempt at making a "REPL" for `jq` using `fzf`. Strictly speaking, it's not a repl in the sense that
the jq compiler is not evaluating code in a loop which it controls itself, as with eg. bash or Python, but technicalities
aside, it behaves like a REPL.

One thing I found to be annoying about the other two tools is that, while you're typing in the jq query, the results of
the previous query are erased. This could probably be addressed to some degree with some "debouncing" logic like in the
official `rg` [example](https://github.com/junegunn/fzf/blob/master/ADVANCED.md#using-fzf-as-interactive-ripgrep-launcher).
However, I also felt that there's an entire half-screen of empty space in `fzf` that's meant to hold lines of input data,
ostensibly for matching. Why couldn't that space hold some useful information? Say, every single key in the document?
Then, not only would exploring the contents of the document be that much easier, my goldfish brain would no longer hamper
me by forgetting which key I was about to add to the query because it would always be right in front of me.

# Implementation

The `jq` query is amazingly simple, considering it took over four hours to figure out:

```jq
def getkeys: [to_entries[] | {(.key): (.value | getkeys?) }];
getkeys
```

This simple, recursive function visits every key in a depth-first order, which is important for this application. The
standard recursive descent operator in `jq`, the `..` filter, is breadth-first. To illustrate the difference, given

```json
{
  "top_key1": {
    "mid_key1": {
      "bottom_key1": 0
    },
    "mid_key2": {
      "bottom_key2": 1
    }
  },
  "top_key2": {
    "mid_key3": {
      "bottom_key3": 2
    },
    "mid_key4": {
      "bottom_key4": 3
    }
  }
}
```

We _want_ this order, as it reflects the path we'll need for a query:

```
top_key1
mid_key1
bottom_key1
mid_key2
bottom_key2
top_key2
mid_key3
bottom_key3
mid_key4
bottom_key4
```

To get this order, the tree must be traversed depth-first. With `..` and a breadth-first ordering, we instead get

```
top_key1
top_key2
mid_key1
mid_key2
mid_key3
mid_key4
bottom_key1
bottom_key2
bottom_key3
bottom_key4
```

which makes it hard to know if the query to get eg. `bottom_key3` should be `.top_key1.mid_key1.bottom_key3` or some other combination of intermediate keys.

# okay, how do I use it?

Install `jq` and `fzf`, of course. Also, install `yq` (which is available from `pip` and many other package managers).

Then, add to your bashrc:

```bash
# Extract all keys recursively (depth-first) with jq from $1, pipe the resulting json
# to yq to get the slightly more compact yaml format, and finally pipe that into fzf,
# with fzf set to run new jq queries against $1.
# This keeps all the keys in view, even while typing a new query and the preview
# disappears.
# Pass '-y' as $1 and a yaml file instead of a json file :D
function jqrepl {
	if [[ $1 = '-y' ]]; then
		q='yq -y'
		mid='cat'
		shift
	else
		q='jq'
		mid='yq -y'
	fi
	$q 'def getkeys: [to_entries[] | {(.key): (.value | getkeys?) }]; getkeys' "$1" \
	| $mid	\
	| fzf	--disabled \
			--print-query \
			--preview "jq --color-output -r {q} $1"
}
```

As the docstring suggests, this also works with YAML documents thanks to `yq`--just pass `-y` as the first argument.

The result will look something like this:
![image](https://github.com/user-attachments/assets/3fa12230-417c-46a3-a064-e294a9ade000)

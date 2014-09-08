+++
date = "2014-09-05T14:12:16-07:00"
title = "JS Delta Walkthrough"

+++

[JS Delta](https://github.com/wala/jsdelta) is a [delta debugger](https://www.st.cs.uni-saarland.de/dd/) for JavaScript-processing tools, primarily written by [Max Schäfer](http://xiemaisi.github.io/).  Delta debugging helps when your tool is failing on a large input, by finding a smaller input that causes the same kind of failure.  It has been widely used since it was [first described by Andreas Zeller](https://www.st.cs.uni-saarland.de/publications/files/zeller-esec-1999.pdf), e.g., to find small programs that cause failures in C compilers (see [Delta](http://delta.tigris.org/) and [C-Reduce](http://embed.cs.utah.edu/creduce/)).

[The JS Delta documentation](https://github.com/wala/jsdelta) actually covers its usage options pretty well, but I thought it would be helpful to see a full walkthrough of the tool in action.  This post shows how JS Delta can be used to find bugs in a very simple analysis.  You can get all the example code [here](https://github.com/msridhar/jsdelta-example); just run `npm install` after cloning.

Our (rather contrived) example analysis aims to do the following: for each occurrence of the form `x.f()` in an input JavaScript program, it prints `X`, i.e., the receiver variable in all caps.  Here is a buggy first version of the analysis, based on [esprima](http://esprima.org/) and [estraverse](https://github.com/Constellation/estraverse):

```javascript
var esprima = require('esprima'),
    estraverse = require('estraverse'),
    fs = require('fs');
var file = process.argv[2];
var ast = esprima.parse(String(fs.readFileSync(file)));
estraverse.traverse(ast, {
  leave: function (node, parent) {
    if (node.type === 'CallExpression')
      console.log(node.callee.object.name.toUpperCase());
  }
});
```

The analysis works for a script `test.js` containing the single statement `x.f();`:

    > node example.js test.js
    X

Now, let's get (too) ambitious and try running the analysis on jQuery:

    > node example.js node_modules/jquery/dist/jquery.js

    /Users/m.sridharan/git-repos/jsdelta-example/example.js:9
        console.log(node.callee.object.name.toUpperCase());
                                      ^
    TypeError: Cannot read property 'name' of undefined
        at Controller.estraverse.traverse.leave (/Users/m.sridharan/git-repos/jsdelta-example/example.js:9:37)
        at Controller.__execute (/Users/m.sridharan/git-repos/jsdelta-example/node_modules/estraverse/estraverse.js:317:31)
        ...

Based on the error message alone, it is a bit hard to diagnose exactly what is wrong with the analysis.  Here, we can use JS Delta to obtain a smaller subset of jQuery that still causes the problem.  To do so, we run jsdelta as follows:

    node node_modules/jsdelta/delta.js --cmd "node example.js" \
    --msg "TypeError: Cannot read property 'name' of undefined" \
    node_modules/jquery/dist/jquery.js

The `--cmd` option gives a shell command to run on each reduced input, and the `--msg` option gives the message that indicates the problem is occurring.  The final option is the original failure-inducing input.  After working for a few seconds, removing parts of jQuery and checking if the error still occurs, JS Delta outputs the following code in `/tmp/tmp0/delta_js_smallest.js`:

```javascript
(function () {
    factory();
}());
```

This is quite a bit smaller than the original jQuery script!  Investigating further, we see that a `CallExpression` does not have an `object` field if no receiver is passed for the call.  Let's patch the analysis code to handle this case:

```javascript
estraverse.traverse(ast, {
    leave: function (node, parent) {
        if (node.type === 'CallExpression' &&
            node.callee.type === 'MemberExpression') {
            console.log(node.callee.object.name.toUpperCase());
        }
    }
});
```

Re-running this analysis on jQuery yields another crash, and running another round of JS Delta yields this input:

```javascript
(function () {
}(function () {
    ({
        pushStack: function () {
            var ret = jQuery.merge(this.constructor());
        }
    });
}));
```

Here, the issue is the `this.constructor()` call, as `this` is represented with a special `ThisExpression` node in the AST.  (It's worth noting at this point that JS Delta typically does not compute a *minimal* failure-inducing input, as that would be too expensive.  There are other transformations we could add that would yield a smaller program for cases like the above, but we haven't implemented them yet.)  Continuing to fix bugs in this manner, we eventually arrive at a working analysis, shown in `example-fixed.js` in the repository.

A couple of further usage notes for JS Delta:

* If you're trying to debug a dynamic analysis, you need to be prepared for some reduced version of the input program to not terminate.  I usually handle this case by prefixing my `--cmd` parameter with [timeout](https://www.gnu.org/software/coreutils/manual/html_node/timeout-invocation.html), e.g., `node delta.js --cmd "timeout foo" ...`.  (On Mac, you'll need to install coreutils to get the `timeout` command, e.g., via [MacPorts](https://www.macports.org/).)
* JS Delta always saves the current smallest input in `delta_js_smallest.js` in its output directory.  You can re-start JS Delta with this file if some reason you have to kill it before it completes a run.  Also, note that running JS Delta multiple times in a row, feeding in the previous smallest input each time, can result in further size reductions.
* JS Delta has often been used to find a small input that causes a static analysis to take an excessively long time to run.  This usage of JS Delta helped us develop and debug the techniques behind [correlation tracking](http://manu.sridharan.net/files/ECOOP12Correlation.pdf) and [dynamic determinacy analysis](http://manu.sridharan.net/files/PLDI13Determinacy.pdf), and it was also used by Andreasen and Møller in developing [their latest techniques](http://cs.au.dk/~amoeller/papers/jquery/paper.pdf) for static analysis of jQuery.  In this use case, its best to avoid defining a "long time" using wall-clock time, as changes in the environment can make execution time vary, making the reduction process flaky.  Instead, use some deterministic measure that correlates with execution time, like the number of fixed-point iterations run thus far.
* Finally, note that since JSON is legal JavaScript, JS Delta can also be used to reduce large JSON inputs.  In general, if you're stuck trying to debug a program crashing on a large input, think about whether you can easily rewrite the program to take the input as a (structured) JSON object.  If so, JS Delta could be helpful in your debugging process.

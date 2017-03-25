Diagrams from a blog post, link to be added later afterwards.

## Diagram Tooling ##

There's a number of JS-based diagramming tools out there.  These are what I
ended up using and why:
* graphviz - Used for non-sequence diagrams and probably any diagram involving
  clustering.  It turns out that you really can't beat graphviz when it comes
  to its support of clusters with cross-cluster edges and the ability to impose
  rank constraints and zero rank weights.  And since viz.js emscripten compiles
  it, it's an option in browsers too.
* http://knsv.github.io/mermaid/ - Used for sequence diagrams and some
  flowcharts.
  Before I fell back to using graphviz more heavily because of its superior
  clustering support over dagre, my default based on its:
  * Goals of supporting many types of documents.
  * Reasonably active core development, plus integration tooling for docbook
    and others.
  * Support of more complicated features, such as subgraphs for flowcharts and
    loops/alternatives in sequence diagrams
  * Reasonable aesthetics with support for different themes.

### How to update diagrams ###

Install pre-requisites:
* graphviz:
  * `apt-get install graphviz` or similar for your platform.
* mermaid:
  * `git clone https://github.com/knsv/mermaid.git`
  * `npm install` in that checkout
  * add `mermaid/bin/mermaid.js` to your path.  I did this by adding a symlink
    to it in my `~/bin` directory that's on my path.

Run "make" and make sure things don't explode.

Then:
* in the `rendered` directory, run `python -m SimpleHTTPServer` so that you can
  connect to http://localhost:8000 (or whatever port you specify as an
  additional argument to that command) and look at the pictures.
* loop:
  * edit files, save them to disk
    * you may want to use the online editors available:
      * https://knsv.github.io/mermaid/live_editor/
  * run make
  * look at pictures.


### Why JS based tools ###

1. A goal to minimize effort required to update diagrams and maximize ability to
   update diagrams.
  * Native applications like inkscape or dia are asking a lot, and some diagram
    applications are platform specific.
  * JS based diagram tools are able to provide interactive editors with
    immediate rendering of the diagram results.
2. Longer term plans to build on searchfox to make it easy to create diagrams
   to aid in exploration, and then from there, to create diagrams to share your
   understanding with others. See
   https://bugzilla.mozilla.org/show_bug.cgi?id=1339243

### Other Tools Considered ###
* JS or transpiled-to-JS
  * https://bramp.github.io/js-sequence-diagrams/ - A classic, but bare bones
    sequence diagram implementation.
  * http://jumly.tmtk.net/ - Nice, fancy-looking (drop shadows!) seqeuence diagram
    mechanism, but largely inactive.  (Also, when considered long ago, the choice
    of coffeescript did not help things.)
  * graph layout algorithms:
    * https://github.com/OpenKieler/klayjs - transpiled from Java via GWT.
      Has interesting layered diagram magic.
* Python
  * blockdiag family: Minimal diagrams, does have an interactive shell web
    service (with rendering on the server), however.
    * http://blockdiag.com/en/blockdiag/index.html - block diagrams
    * http://blockdiag.com/en/seqdiag/index.html - sequence diagrams
    * http://blockdiag.com/en/actdiag/index.html - activity diagrams
    * http://blockdiag.com/en/nwdiag/index.html - network diagrams
* Java - All of these are basically just prior art awareness.
 * http://plantuml.com/http://plantuml.com/
 * https://www.spinellis.gr/umlgraph/

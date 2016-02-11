---
layout: doc_page
---

## Accuracy Plots 

### Accuracy of the QuickSelect Sketch Family

A QuickSelect Sketch, which is the default sketch family, can be constructed with code similar to:  

<div class="highlight"><pre><code class="language-text" data-lang="text">int k = 4096;
UpdateSketch sketch = UpdateSketch.builder().build(k);

// The sketch is fed unique values. Here I use a simple loop for testing.
// u is the number of uniques to be fed to the sketch

long u;
for (int i=0; i&lt;u; i++) {
  sketch.update(i);
}

// After all the unique values have been fed to the sketch, get the estimate of <i>u</i>

double est = sketch.getEstimate();

// An optional rebuild option to reduce the sketch size to <i>k</i> if neccessary 

sketch.rebuild();

// Get the upper and lower bounds. 
// The parameter is the number is the standard deviations from the estimate.

double ub = sketch.getUpperBound(1);
double lb = sketch.getLowerBound(1);
</code></pre></div> 

The accuracy behavior of this QuickSelect Sketch (the default, with no rebuild) will be similar to the following graph:

<img class="doc-img-half" src="{{site.docs_img_dir}}QS4KError.png" alt="QS4KError" />

This type of graph is called a "pitchfork", which is used throughout this documentation to characterize the accuracy of the various sketches.

Pitchfork graphs have these common characteristics:

* The x-axis is Log(base 2) and represents the number of unique values fed into a sketch.
  * Points along the x-axis are evenly spaced along the log axis with the same number of Points Per Octave (PPO) for every multiple of 2 along the x-axis.
This means that the plotted x values form an exponential series of the form <i>2^(gi/PPO)</i>, where <i>gi</i> is the <i>Generating Index</i> for a particular point on the x-axis.
  * A <i>Trial</i> is the estimation result of a single sketch fed <i>x</i> unique values.
  * A <i>Trial Set<i> is the set of results of runing <i>T</i> independent trials at <i>x</i>.

* The y-axis measures the Relative Error (RE) of result estimates returned by the sketches in the trials, where <i>RE = MeasuredValue/TrueValue -1</i> and is plotted as a percent. 
  * An imaginary line drawn vertically from each x-axis point represents the range of error values that result from the Trial Set. 
  However, not all of the error values from all the trials are plotted. 
  Instead, for each Trial Set, the result error values are sorted and then selected quantiles are chosen and then only those y-axis values are plotted. 
  * Connecting the plotted points with the same quantiles form the lines of the graph

* The plotted lines on these graphs are actually the lines of constant quantiles of the distributions of error measured at each x-axis point on the graph.  
* These pitchfork graphs are just two dimensional views of a three-dimensional probability surface that would look something like this: 
<img src="{{site.docs_img_dir}}ErrorSurface2.png" alt="ErrorSurface2" width="150px" />. The cross-sectional slice of this surface is approximately Gaussian like this graph
<img src="{{site.docs_img_dir}}400px-StandardNormalCurve.png" alt="400px-StandardNormalCurve" width="200px" />, 
which has the +/- 1, 2 and 3 standard deviation points on the x-axis marked and the corresponding areas under the curve that represent the associated confidence levels.
* All the pitchfork graphs in this section were generated using the <i>SketchPerformance.java</i> utility that is located in the the <i>test/ ... /sketches/performance</i> package.

The specifics of the above pitchfork graph:

* The sketch type is the Heap QuickSelect Sketch, which is the default Theta UpdateSketch.
* The sketch was configured with <i>k = 4096</i>.
* The x-axis varies from 1024 (2^10) uniques per sketch to 1,048,576 (2^20) uniques per sketch. 
* PPO = 16.
* Trials, T = 4096.
* Wavy colored lines:
  * The black wavy line in the center (at Y = 0%) represents the median of the distribution of error values.  The median sits on top of the mean, which is brown and hard to see. 
  The fact that the mean and median are at zero validates the assertion that the estimates from the sketch are <i>unbiased</i>.
  * The red wavy line is the connected quantiles at +1 <i>Relative Standard Error</i> (RSE), which means ~15.87% of the trial results are above that line.
  * The blue wavy line is the connected quantiles at +2 RSE, which means ~2.3% of the trial results are above that line.
  * The purple wavy line is the connected quantiles at -1 RSE, which means ~15.87% of the trial results are below that line.
  * The green wavy line is the connected quantiles at -2 RSE, which means ~2.3% of the trial results are below that line.
  
This particular graph also illustrates some other subtle points about this particular sketch.

* The saw-toothed variation in the overall shaped is due to the fact that the QuickSelect sketch only updates its internal theta when the hash table fills up, which occurs when the hash table reaches <i>15k/8</i> 
(note that the estimation starts at almost 8K), at which point the sketch uses the QuickSelect algorithm to find the <i>(k + 1)</i><sup>th</sup> order statistic from the cache, 
assigns this value to theta, discards all values above theta, and rebuilds the hash table. 
* Because the number of valid points nearly reaches <i>2k</i> values means that the Relative Error of the sketch nearly reaches <i>1/sqrt(2k)</i>. 

If the user does not want the sketch ever to exceed <i>k</i> values, then there is an optional rebuild method (see code snippet above) that can be used.
This would result in the following graph:

<img class="doc-img-half" src="{{site.docs_img_dir}}QS4KErrorRebuild.png" alt="QS4KErrorRebuild" />

Because of the extra rebuild at the end the full cycle time of the sketch is a little slower and the average accuracy is a little less than without the rebuild. 
This is a tradeoff the user can choose to use or not.

The quantiles for both +/- 1 RSE and +/- 2 RSE establishes the bounds for the 68% and 95.4% confidence levels respectfully.

###Accuracy of the Alpha Sketch Family

Another major sketch family is the Alpha Sketch.  Its pitchfork graph looks like the following:

<img class="doc-img-half" src="{{site.docs_img_dir}}Alpha4KError.png" alt="Alpha4KError" /> 

This sketch was also configured with a size of 4K entries, otherwise the defaults.
The wavy lines have the same interpretation as the wavy lines for the QuickSelect Sketch above.
The short-dashed reference lines are computed at plus and minus <i>1/sqrt(2k)</i>. 
The Alpha Sketch is about 30% more accurate than the QuickSelect Sketch with rebuild for the same value of <i>k</i>. 
However, this improved accuracy can only be obtained when getEstimate() is directly called from the Alpha Sketch. 
After compacting to a Compact Sketch or after any set operation, the accuracy falls back to the standard <i>1/sqrt(k)</i>.
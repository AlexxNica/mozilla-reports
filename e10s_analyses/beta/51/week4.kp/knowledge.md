---
title: E10s Testing for Beta 51 week 4
authors:
- rvitillo
- dzeber
- bmiroglio
tags:
- e10s
- experiment
- add-ons
created_at: 2017-01-10 00:00:00
updated_at: 2017-01-10 10:03:43.326807
tldr: Analysis of e10s experiment for profiles with and without add-ons
thumbnail: images/output_41_2.png
---
# E10s testing for Beta 51 week 4: Main analysis

(This covers data from 2016-12-14 to 2016-12-21 on Beta 51)

## Data processing


```python
import ujson as json
import matplotlib.pyplot as plt
import seaborn
import pandas as pd
import numpy as np
import math
import plotly.plotly as py
import IPython
import pyspark.sql.functions as fun
from pyspark.sql import Row

from __future__ import division
from moztelemetry.spark import get_pings, get_one_ping_per_client, get_pings_properties
from montecarlino import grouped_permutation_test

%pylab inline
IPython.core.pylabtools.figsize(16, 7)
seaborn.set_style('whitegrid')

from operator import add
pd.set_option("display.max_rows", None)
```
    Unable to parse whitelist (/home/hadoop/anaconda2/lib/python2.7/site-packages/moztelemetry/histogram-whitelists.json). Assuming all histograms are acceptable.
    Populating the interactive namespace from numpy and matplotlib



```python
sc.defaultParallelism
```




    320




```python
sc.version
```




    u'1.6.1'




```python
def chi2_distance(xs, ys, eps = 1e-10, normalize = True):
    """ The comparison metric for histograms. """
    histA = xs.sum(axis=0)
    histB = ys.sum(axis=0)
    
    if normalize:
        histA = histA/histA.sum()
        histB = histB/histB.sum()
    
    d = 0.5 * np.sum([((a - b) ** 2) / (a + b + eps)
        for (a, b) in zip(histA, histB)])

    return d

def median_diff(xs, ys):
    return np.median(xs) - np.median(ys)

def make_group_histogram(group_data):
    """ Combine separate client histograms into a single group histogram, normalizing bin counts
        to relative frequencies.       
    """
    ## Check for histograms with 0 counts.
    client_totals = group_data.map(lambda x: x.sum())
    group_data = group_data[client_totals > 0]
    ## Convert frequency counts to relative frequency for each client histogram.
    group_data = group_data.map(lambda x: x/x.sum())
    ## Merge the group's client histograms by adding up the frequencies over all clients
    ## in the group, separately for each bin.
    group_data = group_data.sum()
    ## Convert the merged bin frequencies to relative percentages.
    group_data = 100 * group_data / group_data.sum()
    return group_data
    

def compare_histogram(histogram, e10s_addons, none10s_addons, e10s_std=None, none10s_std=None,
                      include_diff=True, include_diff_in_diff=True, did_separate_plot=True):
    """ Compare an e10s histogram to a non-e10s one, and graph the results.
        
        Plots the two histograms overlaid on the same graph, and prints a p-value
        for testing whether they are different. If 'include_diff' is True, also
        draw a plot of the frequency differences for each bin.
        
        If 'include_diff_in_diff' is True and data is supplied, include a plot of
        differences between addon cohort differences and non-addon cohort differences.
    """
    eTotal = make_group_histogram(e10s_addons)
    nTotal = make_group_histogram(none10s_addons)
    
    if include_diff:
        if include_diff_in_diff and did_separate_plot:
            fig, (ax, diff_ax, diff_diff_ax) = plt.subplots(3, sharex=True, figsize=(16,10), 
                                                            gridspec_kw={"height_ratios": [2,2,1]})
        else:
            fig, (ax, diff_ax) = plt.subplots(2, sharex=True)
    else:
        fig = plt.figure()
        ax = fig.add_subplot(1, 1, 1)
        
    fig.subplots_adjust(hspace=0.3)
    ax2 = ax.twinx()
    width = 0.4
    ylim = max(eTotal.max(), nTotal.max())
        
    eTotal.plot(kind="bar", alpha=0.5, color="green", label="e10s", ax=ax, width=width,
                position=0, ylim=(0, ylim + 1))
    nTotal.plot(kind="bar", alpha=0.5, color="blue", label="non-e10s", ax=ax2, width=width,
                position=1, grid=False, ylim=ax.get_ylim())
    
    ## Combine legend info from both Axes.
    ax_h, ax_l = ax.get_legend_handles_labels()
    ax2_h, ax2_l = ax2.get_legend_handles_labels()
    ax.legend(ax_h + ax2_h, ax_l + ax2_l, loc = 0)
 
    plt.title(histogram)
    ax.xaxis.grid(False)
    ax.set_ylabel("Frequency %")

    if include_diff:
        ## Add a second barplot of the difference in frequency for each bucket.
        #diff_ax = fig.add_subplot(2, 1, 2)
        enDiff = eTotal - nTotal
        
        has_diff_in_diff_data = (e10s_std is not None and len(e10s_std) > 0 and
                                 none10s_std is not None and len(none10s_std) > 0)
        if include_diff_in_diff and has_diff_in_diff_data:
            ## Add bin differences for between e10s/non-e10s for the no-addons cohorts.
            ## The assumption is that the difference between addons cohorts would look the same
            ## if there is no additional effect of having addons.
            eTotal_std = make_group_histogram(e10s_std)
            nTotal_std = make_group_histogram(none10s_std)
            enDiff_std = eTotal_std - nTotal_std
            ylims = (min(enDiff.min(), enDiff_std.min()) - 0.5, max(enDiff.max(), enDiff_std.max()) + 0.5)
            diff_ax2 = diff_ax.twinx()
            
            enDiff.plot(kind="bar", alpha=0.5, color="navy", label="with add-ons", ax=diff_ax, width=width,
                        position=1, ylim=ylims)
            enDiff_std.plot(kind="bar", alpha=0.5, color="gray", label="no add-ons", ax=diff_ax2, width=width,
                        position=0, grid=False, ylim=diff_ax.get_ylim())

            ## Combine legend info from both Axes.
            diff_ax_h, diff_ax_l = diff_ax.get_legend_handles_labels()
            diff_ax2_h, diff_ax2_l = diff_ax2.get_legend_handles_labels()
            leg_h = diff_ax_h + diff_ax2_h
            leg_l = diff_ax_l + diff_ax2_l
            
            if did_separate_plot:
                enDiffDiff = enDiff - enDiff_std
                enDiffDiff.plot(kind="bar", alpha=0.5, color="maroon", ax=diff_diff_ax, ylim=diff_ax.get_ylim())
                diff_diff_ax.xaxis.grid(False)
                diff_diff_ax.set_ylabel("Diff in freq %")
                diff_diff_ax.set_title("Diff between e10s/non diff with add-ons and e10s/non diff without" +
                                      " (with add-ons higher when > 0)")
            
        else:
            if include_diff_in_diff:
                ## We wanted to do the additional comparison, but there wasn't enough data.
                print("\nNo diff-in-diff comparison: one of the standard cohorts has no non-missing observations.")
            enDiff.plot(kind="bar", alpha=0.5, color="navy", label="with add-ons", ax=diff_ax)
            leg_h, leg_l = diff_ax.get_legend_handles_labels()
        
        plt.title("e10s/non-e10s difference (more e10s in bucket when > 0)")
        diff_ax.xaxis.grid(False)
        diff_ax.set_ylabel("Diff in frequency %")
        diff_ax.legend(leg_h, leg_l, loc = 0)
    
    
    # Only display at most 100 tick labels on the x axis.
    xticklabs = plt.gca().get_xticklabels()
    max_x_ticks = 100
    if len(xticklabs) > max_x_ticks:
        step_size = math.ceil(float(len(xticklabs)) / max_x_ticks)
        for i, tl in enumerate(xticklabs):
            if i % step_size != 0:
                tl.set_visible(False)
    plt.show()
    
    ## Compute a p-value for the chi-square distance between the groups' combined histograms.
    pvalue = grouped_permutation_test(chi2_distance, [e10s_addons, none10s_addons], num_samples=100)
    print("The probability that the distributions for {} (with add-ons) are differing by chance is {:.3f}."\
          .format(histogram, pvalue))

def normalize_uptime_hour(frame):
    """ Convert metrics to rates per hour of uptime. """
    frame = frame[frame["payload/simpleMeasurements/totalTime"] > 60]
    frame = 60 * 60 * frame.apply(lambda x: x / frame["payload/simpleMeasurements/totalTime"]) # Metric per hour
    frame.drop('payload/simpleMeasurements/totalTime', axis=1, inplace=True)
    return frame
    
def compare_e10s_count_histograms(pings, cohort_sizes = {}, *histogram_names, **kwargs):
    """ Read multiple count histograms from a collection of pings, and compare e10s/non-e10s for each.
    
        Treats count histograms as scalars for comparison purposes, without distinguishing between
        parent and child processes. Expects a dict containing overall cohort sizes
        for computing sample size proportions.
    """
    properties = histogram_names + ("payload/simpleMeasurements/totalTime", "e10s", "addons")
    frame = pd.DataFrame(get_pings_properties(pings, properties).collect())
    
    e10s = frame[frame["addons"] & frame["e10s"]]
    e10s = normalize_uptime_hour(e10s)
    
    none10s = frame[frame["addons"] & ~frame["e10s"]]
    none10s = normalize_uptime_hour(none10s)
    
    include_diff_in_diff = kwargs.get("include_diff_in_diff", True)
    if include_diff_in_diff:
        e10s_std = normalize_uptime_hour(frame[~frame["addons"] & frame["e10s"]])
        none10s_std = normalize_uptime_hour(frame[~frame["addons"] & ~frame["e10s"]])        
    
    for histogram in histogram_names:
        if histogram not in none10s.columns:
            continue
        
        ## Remove the property path from the histogram name for display purposes.
        hist_name = hist_base_name(histogram)
        if type(hist_name) == list:
            ## Key was given for keyed histogram.
            hist_str = "{}/{}".format(link_to_histogram(hist_name[0]), hist_name[1])
            hist_name = hist_name[0]
        else:
            hist_str = hist_name
        ## Print a header for the block of graphs, including a link to the histogram definition.
        print_with_markdown("Comparison for count histogram {} (with add-ons):".format(hist_str))
        
        e10s_hist = e10s[histogram].dropna()
        non_e10s_hist = none10s[histogram].dropna()
        
        ## Print some information on sample sizes.
        print("{} non-e10s profiles have this histogram.".format(
                sample_size_str(len(non_e10s_hist), cohort_sizes.get("addons-set2a-control"))))
        print("{} e10s profiles have this histogram.".format(
                sample_size_str(len(e10s_hist), cohort_sizes.get("addons-set2a-test"))))
        ## If either group has no data, nothing more to do.
        if len(non_e10s_hist) == 0 or len(e10s_hist) == 0:
            continue
        
        print("")
        compare_scalars(hist_name + " per hour", e10s_hist, non_e10s_hist,
                        e10s_std[histogram].dropna() if include_diff_in_diff else None,
                        none10s_std[histogram].dropna() if include_diff_in_diff else None)
 
def compare_e10s_histograms(pings, cohort_sizes = {}, *histogram_names, **kwargs):
    """ Read multiple histograms from a collection of pings, and compare e10s/non-e10s for each.
    
        Outputs separate comparisons for parent process, child processes, and merged histograms.
        Expects a dict containing overall cohort sizes for computing sample
        size proportions.
    """
    ## Load histogram data from the ping set, separating parent & child processes for e10s.
    frame = pd.DataFrame(get_pings_properties(pings, histogram_names + ("e10s", "addons") , with_processes=True)\
        .collect())
    ## The addons experiment cohorts.
    e10s_addons = frame[frame["addons"] & frame["e10s"]]
    none10s_addons = frame[frame["addons"] & ~frame["e10s"]]
    ## The standard experiment cohorts.
    e10s_std = frame[~frame["addons"] & frame["e10s"]]
    none10s_std = frame[~frame["addons"] & ~frame["e10s"]]
    
    for histogram in histogram_names:
        if histogram not in none10s_addons.columns:
            continue
        
        ## Remove the property path from the histogram name for display purposes.
        hist_name = hist_base_name(histogram)
        if type(hist_name) == list:
            ## Key was given for keyed histogram.
            hist_str = "{}/{}".format(link_to_histogram(hist_name[0]), hist_name[1])
            hist_name = hist_name[0]
        else:
            hist_str = hist_name
        ## Print a header for the block of graphs, including a link to the histogram definition.
        print_with_markdown("Comparison for {} (with add-ons):".format(hist_str))
        
        ## Compare the main histogram for non-e10s against each of 3 for e10s.
        addons_hist_data = {
            "non_e10s": none10s_addons[histogram],
            "e10s_merged": e10s_addons[histogram],
            "e10s_parent": e10s_addons[histogram + "_parent"],
            "e10s_children": e10s_addons[histogram + "_children"]
        }
        for htype in addons_hist_data:
            addons_hist_data[htype] = addons_hist_data[htype].dropna()
        
        ## Print some information on sample sizes.
        sample_sizes = { htype: len(hdata) for htype, hdata in addons_hist_data.iteritems() }
        print("{} non-e10s profiles have this histogram.".format(
                sample_size_str(sample_sizes["non_e10s"], cohort_sizes.get("addons-set2a-control"))))
        print("{} e10s profiles have this histogram.".format(
                sample_size_str(sample_sizes["e10s_merged"], cohort_sizes.get("addons-set2a-test"))))
        ## If either group has no data, nothing more to do.
        if sample_sizes["non_e10s"] == 0 or sample_sizes["e10s_merged"] == 0:
            continue
        
        print("{} e10s profiles have the parent histogram.".format(
                sample_size_str(sample_sizes["e10s_parent"], cohort_sizes.get("addons-set2a-test"))))
        print("{} e10s profiles have the children histogram.".format(
                sample_size_str(sample_sizes["e10s_children"], cohort_sizes.get("addons-set2a-test"))))
        
        has_parent = sample_sizes["e10s_parent"] > 0
        has_children = sample_sizes["e10s_children"] > 0
        
        non_e10s_std_hist = none10s_std[histogram].dropna()
        
        ## Compare merged histograms, unless e10s group has either no parents or no children.
        if has_children and has_parent:
            compare_histogram(hist_name + " (e10s merged)", 
                              addons_hist_data["e10s_merged"],
                              addons_hist_data["non_e10s"],
                              e10s_std[histogram].dropna(),
                              non_e10s_std_hist,
                              **kwargs)
        
        if has_parent:
            compare_histogram(hist_name + " (parent)",
                              addons_hist_data["e10s_parent"],
                              addons_hist_data["non_e10s"],
                              e10s_std[histogram + "_parent"].dropna(),
                              non_e10s_std_hist,
                              **kwargs)

        if has_children:
            compare_histogram(hist_name + " (children)",
                              addons_hist_data["e10s_children"],
                              addons_hist_data["non_e10s"],
                              e10s_std[histogram + "_children"].dropna(),
                              non_e10s_std_hist,
                              **kwargs)

def compare_scalars(metric, e10s_data, non_e10s_data, e10s_std=None, non_e10s_std=None, unit="units"):
    """ Prints info about the median difference between the groups, together with a p-value
        for testing the difference.
        
        Optionally include a string indicating the units the metric is measured in.
        If data is supplied, also print a comparison for non-addons cohorts.
    """
    e10s_data = e10s_data.dropna()
    non_e10s_data = non_e10s_data.dropna()
    if len(e10s_data) == 0 or len(non_e10s_data) == 0:
        print("Cannot run comparison: one of the groups has no non-missing observations.")
        return
    
    print("Comparison for {}{} (with add-ons):\n".format(metric, " ({})".format(unit) if unit != "units" else ""))
    e10s_median = np.median(e10s_data)
    non_e10s_median = np.median(non_e10s_data)
    mdiff = median_diff(e10s_data, non_e10s_data)
    print("- Median with e10s is {:.3g} {} {} median without e10s."\
         .format(
            #abs(mdiff),
            mdiff,
            unit,
            #"higher than" if mdiff >= 0 else "lower than"
            "different from"))
    print("- This is a relative difference of {:.1f}%.".format(float(mdiff) / non_e10s_median * 100))
    print("- E10s group median is {:.4g}, non-e10s group median is {:.4g}.".format(e10s_median, non_e10s_median))
            
    print("\nThe probability of this difference occurring purely by chance is {:.3f}."\
        .format(grouped_permutation_test(median_diff, [e10s_data, non_e10s_data], num_samples=10000)))
    
    if e10s_std is not None and non_e10s_std is not None:
        ## Include a comparison between non-addon cohorts.
        e10s_std = e10s_std.dropna()
        non_e10s_std = non_e10s_std.dropna()
        if len(e10s_std) > 0 and len(non_e10s_std) > 0:
            non_e10s_std_median = np.median(non_e10s_std)
            mdiff_std = median_diff(e10s_std, non_e10s_std)
            print("\nFor cohorts with no add-ons, median with e10s is {:.3g} {} ({:.1f}%) {} median without"\
                 .format(
                    #abs(mdiff_std),
                    mdiff_std,
                    unit,
                    float(mdiff_std) / non_e10s_std_median * 100,
                    #"higher than" if mdiff_std >= 0 else "lower than"
                    "different from"))

    
def link_to_histogram(hist_name):
    """ Create a link to the histogram definition in Markdown. """
    return "[{}](https://dxr.mozilla.org/mozilla-central/search?q={}+file%3AHistograms.json&redirect=true)"\
            .format(hist_name, hist_name)

def hist_base_name(path_to_histogram):
    """ Remove any path components from histogram name.
    
        If histogram is specified as a path in the payload, with separator '/',
        remove everything but the last component (the actual name).
        However, if the histogram is keyed, and specified with a key, return
        [histname, key].
    """
    path_to_histogram = path_to_histogram.rsplit("/")
    if len(path_to_histogram) > 1 and path_to_histogram[-3] == "keyedHistograms":
        ## There was a keyedHistogram name and key given.
        return path_to_histogram[-2:]
    return path_to_histogram[-1]

## Hack to render links in code output.
from IPython.display import Markdown, display
def print_with_markdown(md_text):
    """ Print Markdown text so that it renders correctly in the cell output. """
    display(Markdown(md_text))

def sample_size_str(sample_size, cohort_size=None):
    """ Convert a sample size to a string representation, including a percentage if available. """
    if sample_size == 0:
        return "No"
    if cohort_size:
        if sample_size == cohort_size:
            return "All"
        return "{} ({:.1f}%)".format(sample_size, float(sample_size) / cohort_size * 100)
    return str(sample_size)
```
### Get e10s/non-e10s cohorts for the add-ons experiment

The derived dataset is computed from profiles on Beta 50 who have e10sCohort set. It contains a single record (ping) per client, which is randomly selected from among the client's pings during the date range.


```python
# regenerated data and loaded into telemetry-test-bucket
dataset = sqlContext.read.parquet(
    "s3://telemetry-parquet/e10s_experiment_view/e10s_addons_beta51_cohorts/v20161214_20161221/")
dataset.printSchema()
```
    root
     |-- clientId: string (nullable = false)
     |-- e10sCohort: string (nullable = false)
     |-- creationTimestamp: string (nullable = false)
     |-- submissionDate: string (nullable = false)
     |-- documentId: string (nullable = false)
     |-- sampleId: integer (nullable = false)
     |-- buildId: string (nullable = false)
     |-- simpleMeasurements: string (nullable = false)
     |-- settings: string (nullable = false)
     |-- addons: string (nullable = false)
     |-- system: string (nullable = false)
     |-- build: string (nullable = false)
     |-- threadHangStats: string (nullable = false)
     |-- histograms: string (nullable = false)
     |-- keyedHistograms: string (nullable = false)
     |-- childPayloads: string (nullable = false)
     |-- processes: string (nullable = false)
    


How many records are in the overall dataset?


```python
dataset.count()
```




    2967414



What are the cohorts, and how many clients do we have in each cohort?


```python
%time cohort_counts = dataset.groupby("e10sCohort").count().collect()
dataset_count = sum(map(lambda r: r["count"], cohort_counts))

def cohort_proportions(r):
    prop = r["count"] * 100.0 / dataset_count
    return (r["e10sCohort"], r["count"], "{:.2f}%".format(prop))

print("\nTotal number of clients: {:,}".format(dataset_count))
sorted(map(cohort_proportions, cohort_counts), key = lambda r: r[0])
```
    CPU times: user 8 ms, sys: 0 ns, total: 8 ms
    Wall time: 8.05 s
    
    Total number of clients: 2,967,414






    [(u'addons-set49a-test', 2, '0.00%'),
     (u'addons-set50allmpc-control', 9452, '0.32%'),
     (u'addons-set50allmpc-test', 9010, '0.30%'),
     (u'addons-set51alladdons-control', 504555, '17.00%'),
     (u'addons-set51alladdons-test', 496626, '16.74%'),
     (u'control', 737740, '24.86%'),
     (u'disqualified', 11, '0.00%'),
     (u'disqualified-control', 234581, '7.91%'),
     (u'disqualified-test', 233104, '7.86%'),
     (u'optedIn', 4933, '0.17%'),
     (u'optedOut', 19039, '0.64%'),
     (u'temp-disqualified-ru', 13, '0.00%'),
     (u'test', 714308, '24.07%'),
     (u'unknown', 3976, '0.13%'),
     (u'unsupportedChannel', 64, '0.00%')]




```python
ADDONS_TEST_COHORT = u'addons-set51alladdons-test'
ADDONS_CONTROL_COHORT = u'addons-set51alladdons-control'
```
Restrict to pings belonging to the e10s add-ons experiment. Also include the standard e10s test/control for comparison.


```python
addons_exp_dataset = dataset.filter(\
"e10sCohort in ('%s','%s', 'test', 'control')" % (ADDONS_TEST_COHORT, ADDONS_CONTROL_COHORT))
```
How many clients are left?


```python
addons_exp_dataset.count()
```




    2453229



We want to make sure that the pings tagged into the cohorts satisfy the basic assumptions of the experiment, as this not guaranteed. All add-ons cohort pings should have active add-ons, and e10s should be enabled if and only if the ping belongs to the test cohort.


```python
def e10s_status_check(settings, addons):
    """ Check whether e10s is enabled, and whether there are add-ons. """
    e10sEnabled = json.loads(settings).get("e10sEnabled")
    active_addons = json.loads(addons).get("activeAddons")
    return Row(
        e10s_enabled = bool(e10sEnabled), 
        has_addons = bool(active_addons)
    )

def bad_ping(cohort, settings, addons):
    """ e10s should be enabled iff the profile is in the test cohort, and profiles should have active add-ons
        if they are in the addons cohorts. 
    """
    check_data = e10s_status_check(settings, addons)
    is_bad = cohort.endswith("test") != check_data.e10s_enabled
    if cohort.startswith("addons"):
        is_bad = is_bad or not check_data.has_addons
    return is_bad

## Add a Column to the DF with the outcome of the check.
## This will be used to remove any bad rows after examining them.
from pyspark.sql.types import BooleanType
status_check_udf = fun.udf(bad_ping, BooleanType())
addons_exp_dataset_check = addons_exp_dataset.withColumn("badPing",
    status_check_udf(addons_exp_dataset.e10sCohort, addons_exp_dataset.settings, addons_exp_dataset.addons))
```
If there are any bad pings, describe the problems and remove them from the dataset.


```python
addons_exp_dataset_bad = addons_exp_dataset_check.filter("badPing")\
    .select("e10sCohort", "settings", "addons")\
    .rdd

has_bad = not addons_exp_dataset_bad.isEmpty()
```

```python
if not has_bad:
    print("No issues")
else:
    check_counts = addons_exp_dataset_bad\
        .map(lambda r: (r.e10sCohort, e10s_status_check(r.settings, r.addons)))\
        .countByValue()
    print("Issues:")
    for k, v in check_counts.iteritems():
        print("{}: {}".format(k, v))
```
    Issues:
    (u'addons-set51alladdons-control', Row(e10s_enabled=True, has_addons=True)): 2
    (u'addons-set51alladdons-control', Row(e10s_enabled=False, has_addons=False)): 1396
    (u'addons-set51alladdons-test', Row(e10s_enabled=False, has_addons=True)): 77
    (u'addons-set51alladdons-test', Row(e10s_enabled=True, has_addons=False)): 474
    (u'addons-set51alladdons-test', Row(e10s_enabled=False, has_addons=False)): 1



```python
if has_bad:
    print("\nRemoving these pings from the dataset.")
    addons_exp_dataset = addons_exp_dataset_check.filter("not badPing").drop("badPing")
    print("The dataset now contains {} clients".format(addons_exp_dataset.count()))
```
    
    Removing these pings from the dataset.
    The dataset now contains 2451279 clients


What add-ons are present for the addons cohorts?


```python
def get_active_addon_info(addons_str):
    """ Return a list of currently enabled add-ons in the form (GUID, name, version, isSystem). """
    addons = json.loads(addons_str)
    addons = addons.get("activeAddons", {})
    if not addons:
        return []
    return [(guid, meta.get("name"), meta.get("isSystem"), meta.get('version')) for guid, meta in addons.iteritems()]


def dataset_installed_addons(data, n_top=100):
    """ Extract add-on info from a subset of the main dataset, and generate a table of top add-ons
        with installation counts.
        
        Returns a Pandas DataFrame.
    """
    data_addons = data.select("addons").rdd.map(lambda row: row["addons"])
    data_addons.cache()
    n_in_data = data_addons.count()
    
    ##  Get counts by add-on ID/name/isSystem value.
    addon_counts = data_addons.flatMap(get_active_addon_info)\
        .map(lambda a: (a, 1))\
        .reduceByKey(add)\
        .map(lambda ((guid, name, sys, version), n): (guid, (name, sys, version, n)))
    
    ## Summarize using the most common name and isSystem value.
    top_vals = addon_counts.reduceByKey(lambda a, b: a if a[-1] > b[-1] else b)\
        .map(lambda (guid, (name, sys, version, n)): (guid, (name, sys, version)))
    n_installs = addon_counts.mapValues(lambda (name, sys, version, n): n)\
        .reduceByKey(add)
    addon_info = top_vals.join(n_installs)\
        .map(lambda (guid, ((name, sys, version), n)): {
                "guid": guid,
                "name": name,
                "is_system": sys,
                "version":version,
                "n_installs": n,
                "pct_installed": n / n_in_data * 100
            })\
        .sortBy(lambda info: info["n_installs"], ascending=False)
    
    addon_info_coll = addon_info.collect() if not n_top else addon_info.take(n_top)
    addon_info_table = pd.DataFrame(addon_info_coll)
    addon_info_table = addon_info_table[["guid", "name", "version","is_system", "n_installs", "pct_installed"]]
    ## Number rows from 1.
    addon_info_table.index += 1
    n_addons = addon_info.count()
    data_addons.unpersist()
    return (n_addons, addon_info_table)
```

```python
addons_cohort_num, addons_cohort_table = dataset_installed_addons(
    addons_exp_dataset.filter("e10sCohort like 'addons%'"),
    n_top=100)
print("There were {:,} distinct add-ons installed across the addons cohort.".format(addons_cohort_num))

addons_cohort_table["n_installs"] = addons_cohort_table["n_installs"].map("{:,}".format)
addons_cohort_table["pct_installed"] = addons_cohort_table["pct_installed"].map("{:.2f}".format)
addons_cohort_table
```
    There were 9,692 distinct add-ons installed across the addons cohort.






<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>guid</th>
      <th>name</th>
      <th>version</th>
      <th>is_system</th>
      <th>n_installs</th>
      <th>pct_installed</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>aushelper@mozilla.org</td>
      <td>Application Update Service Helper</td>
      <td>1.0</td>
      <td>True</td>
      <td>997,691</td>
      <td>99.85</td>
    </tr>
    <tr>
      <th>2</th>
      <td>webcompat@mozilla.org</td>
      <td>Web Compat</td>
      <td>1.0</td>
      <td>True</td>
      <td>996,147</td>
      <td>99.69</td>
    </tr>
    <tr>
      <th>3</th>
      <td>e10srollout@mozilla.org</td>
      <td>Multi-process staged rollout</td>
      <td>1.6</td>
      <td>True</td>
      <td>994,338</td>
      <td>99.51</td>
    </tr>
    <tr>
      <th>4</th>
      <td>firefox@getpocket.com</td>
      <td>Pocket</td>
      <td>1.0.5</td>
      <td>True</td>
      <td>993,665</td>
      <td>99.44</td>
    </tr>
    <tr>
      <th>5</th>
      <td>{d10d0bf8-f5b5-c8b4-a8b2-2b9879e08c5d}</td>
      <td>Adblock Plus</td>
      <td>2.8.2</td>
      <td>False</td>
      <td>161,293</td>
      <td>16.14</td>
    </tr>
    <tr>
      <th>6</th>
      <td>{de71f09a-3342-48c5-95c1-4b0f17567554}</td>
      <td>Search for Firefox</td>
      <td>1.2</td>
      <td>False</td>
      <td>78,862</td>
      <td>7.89</td>
    </tr>
    <tr>
      <th>7</th>
      <td>_65Members_@download.fromdoctopdf.com</td>
      <td>FromDocToPDF</td>
      <td>7.102.10.4221</td>
      <td>False</td>
      <td>71,674</td>
      <td>7.17</td>
    </tr>
    <tr>
      <th>8</th>
      <td>light_plugin_ACF0E80077C511E59DED005056C00008@...</td>
      <td>Kaspersky Protection</td>
      <td>4.6.3-15</td>
      <td>False</td>
      <td>59,498</td>
      <td>5.95</td>
    </tr>
    <tr>
      <th>9</th>
      <td>{b9db16a4-6edc-47ec-a1f4-b86292ed211d}</td>
      <td>Video DownloadHelper</td>
      <td>6.1.1</td>
      <td>False</td>
      <td>56,328</td>
      <td>5.64</td>
    </tr>
    <tr>
      <th>10</th>
      <td>helper-sig@savefrom.net</td>
      <td>SaveFrom.net - helper</td>
      <td>6.92</td>
      <td>False</td>
      <td>53,034</td>
      <td>5.31</td>
    </tr>
    <tr>
      <th>11</th>
      <td>_ceMembers_@free.easypdfcombine.com</td>
      <td>EasyPDFCombine</td>
      <td>7.102.10.4117</td>
      <td>False</td>
      <td>40,097</td>
      <td>4.01</td>
    </tr>
    <tr>
      <th>12</th>
      <td>wrc@avast.com</td>
      <td>Avast Online Security</td>
      <td>12.0.88</td>
      <td>False</td>
      <td>39,346</td>
      <td>3.94</td>
    </tr>
    <tr>
      <th>13</th>
      <td>{b9bfaf1c-a63f-47cd-8b9a-29526ced9060}</td>
      <td>Download YouTube Videos as MP4</td>
      <td>1.8.8</td>
      <td>False</td>
      <td>32,302</td>
      <td>3.23</td>
    </tr>
    <tr>
      <th>14</th>
      <td>caa1-aDOiCAxFFMOVIX@jetpack</td>
      <td>goMovix - Movies And More</td>
      <td>0.1.7</td>
      <td>False</td>
      <td>29,581</td>
      <td>2.96</td>
    </tr>
    <tr>
      <th>15</th>
      <td>{4ED1F68A-5463-4931-9384-8FFF5ED91D92}</td>
      <td>McAfee WebAdvisor</td>
      <td>5.0.218.0</td>
      <td>False</td>
      <td>28,496</td>
      <td>2.85</td>
    </tr>
    <tr>
      <th>16</th>
      <td>light_plugin_F6F079488B53499DB99380A7E11A93F6@...</td>
      <td>Kaspersky Protection</td>
      <td>5.0.141-4-20161031140250</td>
      <td>False</td>
      <td>27,556</td>
      <td>2.76</td>
    </tr>
    <tr>
      <th>17</th>
      <td>_dbMembers_@free.getformsonline.com</td>
      <td>GetFormsOnline</td>
      <td>7.102.10.4251</td>
      <td>False</td>
      <td>26,348</td>
      <td>2.64</td>
    </tr>
    <tr>
      <th>18</th>
      <td>_4zMembers_@www.videodownloadconverter.com</td>
      <td>VideoDownloadConverter</td>
      <td>7.102.10.5033</td>
      <td>False</td>
      <td>26,086</td>
      <td>2.61</td>
    </tr>
    <tr>
      <th>19</th>
      <td>firebug@software.joehewitt.com</td>
      <td>Firebug</td>
      <td>2.0.18</td>
      <td>False</td>
      <td>24,840</td>
      <td>2.49</td>
    </tr>
    <tr>
      <th>20</th>
      <td>feca4b87-3be4-43da-a1b1-137c24220968@jetpack</td>
      <td>YouTube Video and Audio Downloader</td>
      <td>0.5.6</td>
      <td>False</td>
      <td>24,367</td>
      <td>2.44</td>
    </tr>
    <tr>
      <th>21</th>
      <td>YoutubeDownloader@PeterOlayev.com</td>
      <td>1-Click YouTube Video Downloader</td>
      <td>2.4.1</td>
      <td>False</td>
      <td>23,715</td>
      <td>2.37</td>
    </tr>
    <tr>
      <th>22</th>
      <td>uBlock0@raymondhill.net</td>
      <td>uBlock Origin</td>
      <td>1.10.2</td>
      <td>False</td>
      <td>21,514</td>
      <td>2.15</td>
    </tr>
    <tr>
      <th>23</th>
      <td>sp@avast.com</td>
      <td>Avast SafePrice</td>
      <td>10.3.5.39</td>
      <td>False</td>
      <td>21,061</td>
      <td>2.11</td>
    </tr>
    <tr>
      <th>24</th>
      <td>artur.dubovoy@gmail.com</td>
      <td>Flash Video Downloader - YouTube HD Download [4K]</td>
      <td>15.0.3</td>
      <td>False</td>
      <td>20,149</td>
      <td>2.02</td>
    </tr>
    <tr>
      <th>25</th>
      <td>{82AF8DCA-6DE9-405D-BD5E-43525BDAD38A}</td>
      <td>Skype</td>
      <td>8.0.0.9103</td>
      <td>False</td>
      <td>20,031</td>
      <td>2.00</td>
    </tr>
    <tr>
      <th>26</th>
      <td>client@anonymox.net</td>
      <td>anonymoX</td>
      <td>2.5.2</td>
      <td>False</td>
      <td>19,341</td>
      <td>1.94</td>
    </tr>
    <tr>
      <th>27</th>
      <td>_8hMembers_@download.allin1convert.com</td>
      <td>Allin1Convert</td>
      <td>7.102.10.3584</td>
      <td>False</td>
      <td>16,818</td>
      <td>1.68</td>
    </tr>
    <tr>
      <th>28</th>
      <td>_dzMembers_@www.pconverter.com</td>
      <td>PConverter</td>
      <td>7.102.10.4851</td>
      <td>False</td>
      <td>16,331</td>
      <td>1.63</td>
    </tr>
    <tr>
      <th>29</th>
      <td>adbhelper@mozilla.org</td>
      <td>ADB Helper</td>
      <td>0.9.0</td>
      <td>False</td>
      <td>15,991</td>
      <td>1.60</td>
    </tr>
    <tr>
      <th>30</th>
      <td>light_plugin_D772DC8D6FAF43A29B25C4EBAA5AD1DE@...</td>
      <td>Kaspersky Protection</td>
      <td>4.6.2-42-20160922074409</td>
      <td>False</td>
      <td>15,548</td>
      <td>1.56</td>
    </tr>
    <tr>
      <th>31</th>
      <td>fxdevtools-adapters@mozilla.org</td>
      <td>Valence</td>
      <td>0.3.5</td>
      <td>False</td>
      <td>15,227</td>
      <td>1.52</td>
    </tr>
    <tr>
      <th>32</th>
      <td>abs@avira.com</td>
      <td>Avira Browser Safety</td>
      <td>2.0.0.10221</td>
      <td>False</td>
      <td>14,843</td>
      <td>1.49</td>
    </tr>
    <tr>
      <th>33</th>
      <td>{DDC359D1-844A-42a7-9AA1-88A850A938A8}</td>
      <td>DownThemAll!</td>
      <td>3.0.8</td>
      <td>False</td>
      <td>14,783</td>
      <td>1.48</td>
    </tr>
    <tr>
      <th>34</th>
      <td>_9pMembers_@free.onlinemapfinder.com</td>
      <td>OnlineMapFinder</td>
      <td>7.102.10.4836</td>
      <td>False</td>
      <td>14,568</td>
      <td>1.46</td>
    </tr>
    <tr>
      <th>35</th>
      <td>{176c8b66-7fc3-4af5-a86b-d0207c456b14}</td>
      <td>Search Powered by Yahoo Engine</td>
      <td>1.0</td>
      <td>False</td>
      <td>14,013</td>
      <td>1.40</td>
    </tr>
    <tr>
      <th>36</th>
      <td>{b9acf540-acba-11e1-8ccb-001fd0e08bd4}</td>
      <td>Easy Youtube Video Downloader Express</td>
      <td>9.11</td>
      <td>False</td>
      <td>13,968</td>
      <td>1.40</td>
    </tr>
    <tr>
      <th>37</th>
      <td>@DownloadManager</td>
      <td>DownloadManager</td>
      <td>0.2.1</td>
      <td>False</td>
      <td>13,722</td>
      <td>1.37</td>
    </tr>
    <tr>
      <th>38</th>
      <td>firefox@mega.co.nz</td>
      <td>MEGA</td>
      <td>3.6.19</td>
      <td>False</td>
      <td>13,244</td>
      <td>1.33</td>
    </tr>
    <tr>
      <th>39</th>
      <td>ar1er-ewrgfdgomusix@jetpack</td>
      <td>goMusix</td>
      <td>1.0.6</td>
      <td>False</td>
      <td>13,146</td>
      <td>1.32</td>
    </tr>
    <tr>
      <th>40</th>
      <td>_agMembers_@free.premierdownloadmanager.com</td>
      <td>PremierDownloadManager</td>
      <td>7.102.10.4846</td>
      <td>False</td>
      <td>12,662</td>
      <td>1.27</td>
    </tr>
    <tr>
      <th>41</th>
      <td>{e4a8a97b-f2ed-450b-b12d-ee082ba24781}</td>
      <td>Greasemonkey</td>
      <td>3.9</td>
      <td>False</td>
      <td>12,606</td>
      <td>1.26</td>
    </tr>
    <tr>
      <th>42</th>
      <td>www.facebook.com@services.mozilla.org</td>
      <td>Facebook</td>
      <td>2</td>
      <td>None</td>
      <td>12,198</td>
      <td>1.22</td>
    </tr>
    <tr>
      <th>43</th>
      <td>_paMembers_@www.filmfanatic.com</td>
      <td>FilmFanatic</td>
      <td>7.102.10.4163</td>
      <td>False</td>
      <td>12,159</td>
      <td>1.22</td>
    </tr>
    <tr>
      <th>44</th>
      <td>_gtMembers_@free.gamingwonderland.com</td>
      <td>GamingWonderland</td>
      <td>7.102.10.4263</td>
      <td>False</td>
      <td>11,593</td>
      <td>1.16</td>
    </tr>
    <tr>
      <th>45</th>
      <td>avg@toolbar</td>
      <td>AVG Web TuneUp</td>
      <td>4.3.6.255</td>
      <td>False</td>
      <td>10,868</td>
      <td>1.09</td>
    </tr>
    <tr>
      <th>46</th>
      <td>vb@yandex.ru</td>
      <td>Визуальные закладки</td>
      <td>2.31.3</td>
      <td>False</td>
      <td>10,291</td>
      <td>1.03</td>
    </tr>
    <tr>
      <th>47</th>
      <td>_fsMembers_@free.pdfconverterhq.com</td>
      <td>PDFConverterHQ</td>
      <td>7.102.10.4849</td>
      <td>False</td>
      <td>10,246</td>
      <td>1.03</td>
    </tr>
    <tr>
      <th>48</th>
      <td>jid0-GXjLLfbCoAx0LcltEdFrEkQdQPI@jetpack</td>
      <td>Awesome Screenshot - Capture, Annotate &amp; More</td>
      <td>3.0.14</td>
      <td>False</td>
      <td>10,168</td>
      <td>1.02</td>
    </tr>
    <tr>
      <th>49</th>
      <td>LVD-SAE@iacsearchandmedia.com</td>
      <td>iLivid</td>
      <td>8.5</td>
      <td>False</td>
      <td>10,126</td>
      <td>1.01</td>
    </tr>
    <tr>
      <th>50</th>
      <td>_6xMembers_@www.readingfanatic.com</td>
      <td>ReadingFanatic</td>
      <td>7.102.10.4914</td>
      <td>False</td>
      <td>9,912</td>
      <td>0.99</td>
    </tr>
    <tr>
      <th>51</th>
      <td>{bee6eb20-01e0-ebd1-da83-080329fb9a3a}</td>
      <td>Flash and Video Download</td>
      <td>2.03</td>
      <td>False</td>
      <td>9,521</td>
      <td>0.95</td>
    </tr>
    <tr>
      <th>52</th>
      <td>@mysmartprice-ff</td>
      <td>MySmartPrice</td>
      <td>0.0.6</td>
      <td>False</td>
      <td>9,461</td>
      <td>0.95</td>
    </tr>
    <tr>
      <th>53</th>
      <td>mozilla_cc2@internetdownloadmanager.com</td>
      <td>IDM integration</td>
      <td>6.23.19</td>
      <td>False</td>
      <td>9,394</td>
      <td>0.94</td>
    </tr>
    <tr>
      <th>54</th>
      <td>87677a2c52b84ad3a151a4a72f5bd3c4@jetpack</td>
      <td>Grammarly for Firefox</td>
      <td>8.698.584</td>
      <td>False</td>
      <td>8,564</td>
      <td>0.86</td>
    </tr>
    <tr>
      <th>55</th>
      <td>jid1-YcMV6ngYmQRA2w@jetpack</td>
      <td>Pin It button</td>
      <td>1.37.9</td>
      <td>False</td>
      <td>8,549</td>
      <td>0.86</td>
    </tr>
    <tr>
      <th>56</th>
      <td>{58d735b4-9d6c-4e37-b146-7b9f7e79e318}</td>
      <td>Findwide Search Engine</td>
      <td>1.6</td>
      <td>False</td>
      <td>8,543</td>
      <td>0.85</td>
    </tr>
    <tr>
      <th>57</th>
      <td>anttoolbar@ant.com</td>
      <td>Ant Video Downloader</td>
      <td>2.4.7.47</td>
      <td>False</td>
      <td>8,337</td>
      <td>0.83</td>
    </tr>
    <tr>
      <th>58</th>
      <td>sovetnik@metabar.ru</td>
      <td>Советник Яндекс.Маркета</td>
      <td>3.1.4.90</td>
      <td>False</td>
      <td>8,279</td>
      <td>0.83</td>
    </tr>
    <tr>
      <th>59</th>
      <td>@testpilot-addon</td>
      <td>Test Pilot</td>
      <td>0.9.1-dev-e42d9cb</td>
      <td>False</td>
      <td>7,843</td>
      <td>0.78</td>
    </tr>
    <tr>
      <th>60</th>
      <td>WebProtection@360safe.com</td>
      <td>360 Internet Protection</td>
      <td>5.0.0.1005</td>
      <td>False</td>
      <td>7,723</td>
      <td>0.77</td>
    </tr>
    <tr>
      <th>61</th>
      <td>ERAIL.IN.FFPLUGIN@jetpack</td>
      <td>ERail Plugin for Firefox</td>
      <td>6.0.rev142</td>
      <td>False</td>
      <td>7,659</td>
      <td>0.77</td>
    </tr>
    <tr>
      <th>62</th>
      <td>adblockpopups@jessehakanen.net</td>
      <td>Adblock Plus Pop-up Addon</td>
      <td>0.9.2.1-signed.1-signed</td>
      <td>False</td>
      <td>7,623</td>
      <td>0.76</td>
    </tr>
    <tr>
      <th>63</th>
      <td>yasearch@yandex.ru</td>
      <td>Yandex Elements</td>
      <td>8.20.4</td>
      <td>False</td>
      <td>7,447</td>
      <td>0.75</td>
    </tr>
    <tr>
      <th>64</th>
      <td>firefox@ghostery.com</td>
      <td>Ghostery</td>
      <td>7.1.1.5</td>
      <td>False</td>
      <td>7,044</td>
      <td>0.70</td>
    </tr>
    <tr>
      <th>65</th>
      <td>{19503e42-ca3c-4c27-b1e2-9cdb2170ee34}</td>
      <td>FlashGot</td>
      <td>1.5.6.14</td>
      <td>False</td>
      <td>6,974</td>
      <td>0.70</td>
    </tr>
    <tr>
      <th>66</th>
      <td>{C1A2A613-35F1-4FCF-B27F-2840527B6556}</td>
      <td>Norton Security Toolbar</td>
      <td>2016.8.1.9</td>
      <td>False</td>
      <td>6,647</td>
      <td>0.67</td>
    </tr>
    <tr>
      <th>67</th>
      <td>bingsearch.full@microsoft.com</td>
      <td>Bing Search</td>
      <td>1.0.0.8</td>
      <td>False</td>
      <td>6,622</td>
      <td>0.66</td>
    </tr>
    <tr>
      <th>68</th>
      <td>_b7Members_@free.mytransitguide.com</td>
      <td>MyTransitGuide</td>
      <td>7.102.10.4812</td>
      <td>False</td>
      <td>6,578</td>
      <td>0.66</td>
    </tr>
    <tr>
      <th>69</th>
      <td>_64Members_@www.televisionfanatic.com</td>
      <td>TelevisionFanatic</td>
      <td>7.102.10.4968</td>
      <td>False</td>
      <td>6,408</td>
      <td>0.64</td>
    </tr>
    <tr>
      <th>70</th>
      <td>@Email</td>
      <td>Email</td>
      <td>4.0.12</td>
      <td>False</td>
      <td>6,382</td>
      <td>0.64</td>
    </tr>
    <tr>
      <th>71</th>
      <td>firefox@zenmate.com</td>
      <td>ZenMate Security, Privacy &amp; Unblock VPN</td>
      <td>5.9.0</td>
      <td>False</td>
      <td>6,329</td>
      <td>0.63</td>
    </tr>
    <tr>
      <th>72</th>
      <td>_9tMembers_@free.internetspeedtracker.com</td>
      <td>InternetSpeedTracker</td>
      <td>7.102.10.4339</td>
      <td>False</td>
      <td>6,184</td>
      <td>0.62</td>
    </tr>
    <tr>
      <th>73</th>
      <td>jid1-HAV2inXAnQPIeA@jetpack</td>
      <td>YouTube™ Flash® Player</td>
      <td>1.7.1</td>
      <td>False</td>
      <td>5,976</td>
      <td>0.60</td>
    </tr>
    <tr>
      <th>74</th>
      <td>info@youtube-mp3.org</td>
      <td>YouTube mp3</td>
      <td>1.0.9.1-signed.1-signed</td>
      <td>False</td>
      <td>5,904</td>
      <td>0.59</td>
    </tr>
    <tr>
      <th>75</th>
      <td>ffext_basicvideoext@startpage24</td>
      <td>Video Downloader professional</td>
      <td>1.97.37.1-signed.1-signed</td>
      <td>False</td>
      <td>5,887</td>
      <td>0.59</td>
    </tr>
    <tr>
      <th>76</th>
      <td>MUB-SAE@iacsearchandmedia.com</td>
      <td>Music Box</td>
      <td>8.7</td>
      <td>False</td>
      <td>5,705</td>
      <td>0.57</td>
    </tr>
    <tr>
      <th>77</th>
      <td>{73a6fe31-595d-460b-a920-fcc0f8843232}</td>
      <td>NoScript</td>
      <td>2.9.5.2</td>
      <td>False</td>
      <td>5,611</td>
      <td>0.56</td>
    </tr>
    <tr>
      <th>78</th>
      <td>_gcMembers_@www.weatherblink.com</td>
      <td>WeatherBlink</td>
      <td>7.38.8.56523</td>
      <td>False</td>
      <td>5,545</td>
      <td>0.55</td>
    </tr>
    <tr>
      <th>79</th>
      <td>_4jMembers_@www.radiorage.com</td>
      <td>RadioRage</td>
      <td>7.102.10.4916</td>
      <td>False</td>
      <td>5,532</td>
      <td>0.55</td>
    </tr>
    <tr>
      <th>80</th>
      <td>_dqMembers_@www.downspeedtest.com</td>
      <td>DownSpeedTest</td>
      <td>7.102.10.3827</td>
      <td>False</td>
      <td>5,525</td>
      <td>0.55</td>
    </tr>
    <tr>
      <th>81</th>
      <td>{f3bd3dd2-2888-44c5-91a2-2caeb33fb898}</td>
      <td>YouTube Flash Video Player</td>
      <td>50.0</td>
      <td>False</td>
      <td>5,109</td>
      <td>0.51</td>
    </tr>
    <tr>
      <th>82</th>
      <td>tvplusnewtab-the-extension1@mozilla.com</td>
      <td>tvplusnewtab Extension</td>
      <td>0.1.5</td>
      <td>False</td>
      <td>5,070</td>
      <td>0.51</td>
    </tr>
    <tr>
      <th>83</th>
      <td>mg.mail.yahoo.com@services.mozilla.org</td>
      <td>Yahoo Mail</td>
      <td>1.0</td>
      <td>None</td>
      <td>4,969</td>
      <td>0.50</td>
    </tr>
    <tr>
      <th>84</th>
      <td>translator@zoli.bod</td>
      <td>Google Translator for Firefox</td>
      <td>2.1.0.5.1.1-signed</td>
      <td>False</td>
      <td>4,950</td>
      <td>0.50</td>
    </tr>
    <tr>
      <th>85</th>
      <td>{a38384b3-2d1d-4f36-bc22-0f7ae402bcd7}</td>
      <td>Визуальные закладки @Mail.Ru</td>
      <td>1.0.0.51</td>
      <td>False</td>
      <td>4,856</td>
      <td>0.49</td>
    </tr>
    <tr>
      <th>86</th>
      <td>_1eMembers_@www.videoscavenger.com</td>
      <td>VideoScavenger</td>
      <td>7.38.8.45273</td>
      <td>False</td>
      <td>4,774</td>
      <td>0.48</td>
    </tr>
    <tr>
      <th>87</th>
      <td>homepage@mail.ru</td>
      <td>Домашняя страница Mail.Ru</td>
      <td>1.0.2</td>
      <td>False</td>
      <td>4,668</td>
      <td>0.47</td>
    </tr>
    <tr>
      <th>88</th>
      <td>paulsaintuzb@gmail.com</td>
      <td>Youtube Downloader - 4K Download</td>
      <td>8.2.1</td>
      <td>False</td>
      <td>4,620</td>
      <td>0.46</td>
    </tr>
    <tr>
      <th>89</th>
      <td>_69Members_@www.packagetracer.com</td>
      <td>PackageTracer</td>
      <td>7.102.10.4831</td>
      <td>False</td>
      <td>4,605</td>
      <td>0.46</td>
    </tr>
    <tr>
      <th>90</th>
      <td>{7b8a500a-a464-4624-bd4f-73eaafe0f766}</td>
      <td>Video AdBlock</td>
      <td>3.0</td>
      <td>False</td>
      <td>4,546</td>
      <td>0.45</td>
    </tr>
    <tr>
      <th>91</th>
      <td>search@mail.ru</td>
      <td>Поиск@Mail.Ru</td>
      <td>1.0.7</td>
      <td>False</td>
      <td>4,516</td>
      <td>0.45</td>
    </tr>
    <tr>
      <th>92</th>
      <td>k7srff_enUS@k7computing.com</td>
      <td>K7 WebProtection</td>
      <td>2.4</td>
      <td>False</td>
      <td>4,478</td>
      <td>0.45</td>
    </tr>
    <tr>
      <th>93</th>
      <td>jid1-q4sG8pYhq8KGHs@jetpack</td>
      <td>AdBlocker for YouTube™</td>
      <td>0.2.5</td>
      <td>False</td>
      <td>4,428</td>
      <td>0.44</td>
    </tr>
    <tr>
      <th>94</th>
      <td>{a0d7ccb3-214d-498b-b4aa-0e8fda9a7bf7}</td>
      <td>WOT</td>
      <td>20151208</td>
      <td>False</td>
      <td>4,417</td>
      <td>0.44</td>
    </tr>
    <tr>
      <th>95</th>
      <td>{170503FA-3349-4F17-BC86-001888A5C8E2}</td>
      <td>Youtube Best Video Downloader 2</td>
      <td>6.2</td>
      <td>False</td>
      <td>4,325</td>
      <td>0.43</td>
    </tr>
    <tr>
      <th>96</th>
      <td>support@lastpass.com</td>
      <td>LastPass</td>
      <td>3.3.1</td>
      <td>False</td>
      <td>4,316</td>
      <td>0.43</td>
    </tr>
    <tr>
      <th>97</th>
      <td>_e5Members_@www.productivityboss.com</td>
      <td>ProductivityBoss</td>
      <td>7.38.8.46590</td>
      <td>False</td>
      <td>4,260</td>
      <td>0.43</td>
    </tr>
    <tr>
      <th>98</th>
      <td>vdpure@link64</td>
      <td>Youtube and more - Easy Video Downloader</td>
      <td>1.97.43</td>
      <td>False</td>
      <td>4,247</td>
      <td>0.43</td>
    </tr>
    <tr>
      <th>99</th>
      <td>{5384767E-00D9-40E9-B72F-9CC39D655D6F}</td>
      <td>EPUBReader</td>
      <td>1.5.0.9</td>
      <td>False</td>
      <td>4,142</td>
      <td>0.41</td>
    </tr>
    <tr>
      <th>100</th>
      <td>jetpack-extension@dashlane.com</td>
      <td>Dashlane</td>
      <td>4.2.3</td>
      <td>False</td>
      <td>4,129</td>
      <td>0.41</td>
    </tr>
  </tbody>
</table>
</div>



What add-ons are present in the standard (non-addons) cohorts, if any?


```python
std_cohort_num, std_cohort_table = dataset_installed_addons(
    addons_exp_dataset.filter("e10sCohort in ('test', 'control')"),
    n_top=100)
print("There were {:,} distinct add-ons installed across the standard cohort.".format(std_cohort_num))

std_cohort_table["n_installs"] = std_cohort_table["n_installs"].map("{:,}".format)
std_cohort_table["pct_installed"] = std_cohort_table["pct_installed"].map("{:.2f}".format)
std_cohort_table
```
    There were 1,027 distinct add-ons installed across the standard cohort.






<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>guid</th>
      <th>name</th>
      <th>version</th>
      <th>is_system</th>
      <th>n_installs</th>
      <th>pct_installed</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>aushelper@mozilla.org</td>
      <td>Application Update Service Helper</td>
      <td>1.0</td>
      <td>True</td>
      <td>1,427,279</td>
      <td>98.29</td>
    </tr>
    <tr>
      <th>2</th>
      <td>webcompat@mozilla.org</td>
      <td>Web Compat</td>
      <td>1.0</td>
      <td>True</td>
      <td>1,425,821</td>
      <td>98.19</td>
    </tr>
    <tr>
      <th>3</th>
      <td>e10srollout@mozilla.org</td>
      <td>Multi-process staged rollout</td>
      <td>1.6</td>
      <td>True</td>
      <td>1,424,642</td>
      <td>98.11</td>
    </tr>
    <tr>
      <th>4</th>
      <td>firefox@getpocket.com</td>
      <td>Pocket</td>
      <td>1.0.5</td>
      <td>True</td>
      <td>1,424,427</td>
      <td>98.10</td>
    </tr>
    <tr>
      <th>5</th>
      <td>www.facebook.com@services.mozilla.org</td>
      <td>Facebook</td>
      <td>2</td>
      <td>None</td>
      <td>6,248</td>
      <td>0.43</td>
    </tr>
    <tr>
      <th>6</th>
      <td>firefox-hotfix@mozilla.org</td>
      <td>Firefox Hotfix</td>
      <td>20160826.01</td>
      <td>False</td>
      <td>4,697</td>
      <td>0.32</td>
    </tr>
    <tr>
      <th>7</th>
      <td>mg.mail.yahoo.com@services.mozilla.org</td>
      <td>Yahoo Mail</td>
      <td>1.0</td>
      <td>None</td>
      <td>2,874</td>
      <td>0.20</td>
    </tr>
    <tr>
      <th>8</th>
      <td>plus.google.com@services.mozilla.org</td>
      <td>Google+</td>
      <td></td>
      <td>None</td>
      <td>2,258</td>
      <td>0.16</td>
    </tr>
    <tr>
      <th>9</th>
      <td>content_blocker@kaspersky.com</td>
      <td>Dangerous Websites Blocker</td>
      <td>4.0.10.15</td>
      <td>False</td>
      <td>2,199</td>
      <td>0.15</td>
    </tr>
    <tr>
      <th>10</th>
      <td>loop@mozilla.org</td>
      <td>Firefox Hello</td>
      <td>1.2.6</td>
      <td>True</td>
      <td>1,561</td>
      <td>0.11</td>
    </tr>
    <tr>
      <th>11</th>
      <td>online_banking@kaspersky.com</td>
      <td>Safe Money</td>
      <td>4.0.10.15</td>
      <td>False</td>
      <td>1,283</td>
      <td>0.09</td>
    </tr>
    <tr>
      <th>12</th>
      <td>virtual_keyboard@kaspersky.com</td>
      <td>Virtual Keyboard</td>
      <td>4.0.10.15</td>
      <td>False</td>
      <td>1,280</td>
      <td>0.09</td>
    </tr>
    <tr>
      <th>13</th>
      <td>anti_banner@kaspersky.com</td>
      <td>Anti-Banner</td>
      <td>4.0.10.15</td>
      <td>False</td>
      <td>1,277</td>
      <td>0.09</td>
    </tr>
    <tr>
      <th>14</th>
      <td>url_advisor@kaspersky.com</td>
      <td>Kaspersky URL Advisor</td>
      <td>4.0.10.15</td>
      <td>False</td>
      <td>1,276</td>
      <td>0.09</td>
    </tr>
    <tr>
      <th>15</th>
      <td>light_plugin_F6F079488B53499DB99380A7E11A93F6@...</td>
      <td>Kaspersky Protection</td>
      <td>5.0.141-4-20161031140250</td>
      <td>False</td>
      <td>1,165</td>
      <td>0.08</td>
    </tr>
    <tr>
      <th>16</th>
      <td>light_plugin_ACF0E80077C511E59DED005056C00008@...</td>
      <td>Kaspersky Protection</td>
      <td>4.6.3-15</td>
      <td>False</td>
      <td>942</td>
      <td>0.06</td>
    </tr>
    <tr>
      <th>17</th>
      <td>{C7AE725D-FA5C-4027-BB4C-787EF9F8248A}</td>
      <td>RelevantKnowledge</td>
      <td>1.0.0.4</td>
      <td>False</td>
      <td>845</td>
      <td>0.06</td>
    </tr>
    <tr>
      <th>18</th>
      <td>{95E84BD3-3604-4AAC-B2CA-D9AC3E55B64B}</td>
      <td>Adblocker for Youtube™</td>
      <td>2.0.0.78</td>
      <td>True</td>
      <td>726</td>
      <td>0.05</td>
    </tr>
    <tr>
      <th>19</th>
      <td>virtual_keyboard_294FF26A1D5B455495946778FDE7C...</td>
      <td>Virtual Keyboard</td>
      <td>4.5.3.8</td>
      <td>False</td>
      <td>699</td>
      <td>0.05</td>
    </tr>
    <tr>
      <th>20</th>
      <td>content_blocker_6418E0D362104DADA084DC312DFA8A...</td>
      <td>Dangerous Websites Blocker</td>
      <td>4.5.3.8</td>
      <td>False</td>
      <td>699</td>
      <td>0.05</td>
    </tr>
    <tr>
      <th>21</th>
      <td>{82AF8DCA-6DE9-405D-BD5E-43525BDAD38A}</td>
      <td>Skype</td>
      <td>8.0.0.9103</td>
      <td>False</td>
      <td>672</td>
      <td>0.05</td>
    </tr>
    <tr>
      <th>22</th>
      <td>{d10d0bf8-f5b5-c8b4-a8b2-2b9879e08c5d}</td>
      <td>Adblock Plus</td>
      <td>2.8.2</td>
      <td>False</td>
      <td>545</td>
      <td>0.04</td>
    </tr>
    <tr>
      <th>23</th>
      <td>twitter.com@services.mozilla.org</td>
      <td>Twitter</td>
      <td></td>
      <td>None</td>
      <td>448</td>
      <td>0.03</td>
    </tr>
    <tr>
      <th>24</th>
      <td>online_banking_69A4E213815F42BD863D889007201D8...</td>
      <td>Safe Money</td>
      <td>4.5.3.8</td>
      <td>False</td>
      <td>438</td>
      <td>0.03</td>
    </tr>
    <tr>
      <th>25</th>
      <td>{4ED1F68A-5463-4931-9384-8FFF5ED91D92}</td>
      <td>McAfee WebAdvisor</td>
      <td>5.0.218.0</td>
      <td>False</td>
      <td>408</td>
      <td>0.03</td>
    </tr>
    <tr>
      <th>26</th>
      <td>uBlock0@raymondhill.net</td>
      <td>uBlock Origin</td>
      <td>1.10.2</td>
      <td>False</td>
      <td>383</td>
      <td>0.03</td>
    </tr>
    <tr>
      <th>27</th>
      <td>{de71f09a-3342-48c5-95c1-4b0f17567554}</td>
      <td>Search for Firefox</td>
      <td>1.2</td>
      <td>False</td>
      <td>345</td>
      <td>0.02</td>
    </tr>
    <tr>
      <th>28</th>
      <td>amcontextmenu@loucypher</td>
      <td>Fast search</td>
      <td>0.4.2.1-signed.1-signed</td>
      <td>False</td>
      <td>334</td>
      <td>0.02</td>
    </tr>
    <tr>
      <th>29</th>
      <td>www.linkedin.com@services.mozilla.org</td>
      <td>LinkedIn</td>
      <td></td>
      <td>None</td>
      <td>317</td>
      <td>0.02</td>
    </tr>
    <tr>
      <th>30</th>
      <td>mail.google.com@services.mozilla.org</td>
      <td>Gmail</td>
      <td></td>
      <td>None</td>
      <td>247</td>
      <td>0.02</td>
    </tr>
    <tr>
      <th>31</th>
      <td>wrc@avast.com</td>
      <td>Avast Online Security</td>
      <td>12.0.88</td>
      <td>False</td>
      <td>247</td>
      <td>0.02</td>
    </tr>
    <tr>
      <th>32</th>
      <td>{b9db16a4-6edc-47ec-a1f4-b86292ed211d}</td>
      <td>Video DownloadHelper</td>
      <td>6.1.1</td>
      <td>False</td>
      <td>231</td>
      <td>0.02</td>
    </tr>
    <tr>
      <th>33</th>
      <td>_65Members_@download.fromdoctopdf.com</td>
      <td>FromDocToPDF</td>
      <td>7.102.10.4221</td>
      <td>False</td>
      <td>207</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>34</th>
      <td>sp@avast.com</td>
      <td>Avast SafePrice</td>
      <td>10.3.5.39</td>
      <td>False</td>
      <td>197</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>35</th>
      <td>googletestNT@mozillaonline.com</td>
      <td>Firefox Homepage</td>
      <td>0.10.43</td>
      <td>True</td>
      <td>189</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>36</th>
      <td>client@anonymox.net</td>
      <td>anonymoX</td>
      <td>2.5.2</td>
      <td>False</td>
      <td>175</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>37</th>
      <td>www.tumblr.com@services.mozilla.org</td>
      <td>Tumblr</td>
      <td>1</td>
      <td>None</td>
      <td>172</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>38</th>
      <td>_ceMembers_@free.easypdfcombine.com</td>
      <td>EasyPDFCombine</td>
      <td>7.102.10.4117</td>
      <td>False</td>
      <td>149</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>39</th>
      <td>{6E727987-C8EA-44DA-8749-310C0FBE3C3E}</td>
      <td>TSearch</td>
      <td>2.0.0.35</td>
      <td>True</td>
      <td>133</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>40</th>
      <td>{b9bfaf1c-a63f-47cd-8b9a-29526ced9060}</td>
      <td>Download YouTube Videos as MP4</td>
      <td>1.8.8</td>
      <td>False</td>
      <td>130</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>41</th>
      <td>www.ok.ru@services.mozilla.org</td>
      <td>Odnoklassniki</td>
      <td></td>
      <td>None</td>
      <td>119</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>42</th>
      <td>firebug@software.joehewitt.com</td>
      <td>Firebug</td>
      <td>2.0.18</td>
      <td>False</td>
      <td>119</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>43</th>
      <td>content_blocker_663BE84DBCC949E88C7600F63CA7F0...</td>
      <td>Dangerous Websites Blocker</td>
      <td>4.5.1.379</td>
      <td>False</td>
      <td>115</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>44</th>
      <td>virtual_keyboard_07402848C2F6470194F131B0F3DE0...</td>
      <td>Virtual Keyboard</td>
      <td>4.5.1.379</td>
      <td>False</td>
      <td>115</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>45</th>
      <td>vk.com@services.mozilla.org</td>
      <td>ВКонтакте</td>
      <td>1</td>
      <td>None</td>
      <td>114</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>46</th>
      <td>artur.dubovoy@gmail.com</td>
      <td>Flash Video Downloader - YouTube HD Download [4K]</td>
      <td>15.0.5</td>
      <td>False</td>
      <td>95</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>47</th>
      <td>{DDC359D1-844A-42a7-9AA1-88A850A938A8}</td>
      <td>DownThemAll!</td>
      <td>3.0.8</td>
      <td>False</td>
      <td>87</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>48</th>
      <td>feca4b87-3be4-43da-a1b1-137c24220968@jetpack</td>
      <td>YouTube Video and Audio Downloader</td>
      <td>0.5.6</td>
      <td>False</td>
      <td>86</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>49</th>
      <td>jid0-GXjLLfbCoAx0LcltEdFrEkQdQPI@jetpack</td>
      <td>Awesome Screenshot - Capture, Annotate &amp; More</td>
      <td>3.0.14</td>
      <td>False</td>
      <td>85</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>50</th>
      <td>online_banking_08806E753BE44495B44E90AA2513BDC...</td>
      <td>Safe Money</td>
      <td>4.5.1.379</td>
      <td>False</td>
      <td>85</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>51</th>
      <td>{746505DC-0E21-4667-97F8-72EA6BCF5EEF}</td>
      <td>Shopper-Pro</td>
      <td>1.0.0.4</td>
      <td>False</td>
      <td>84</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>52</th>
      <td>ubufox@ubuntu.com</td>
      <td>Ubuntu Modifications</td>
      <td>3.2</td>
      <td>False</td>
      <td>82</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>53</th>
      <td>{C1A2A613-35F1-4FCF-B27F-2840527B6556}</td>
      <td>Norton Security Toolbar</td>
      <td>2016.8.1.9</td>
      <td>False</td>
      <td>82</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>54</th>
      <td>@Converter</td>
      <td>Converter</td>
      <td>4.1.0</td>
      <td>False</td>
      <td>79</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>55</th>
      <td>{7b1bf0b6-a1b9-42b0-b75d-252036438bdc}</td>
      <td>YouTube High Definition</td>
      <td>50.0</td>
      <td>False</td>
      <td>76</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>56</th>
      <td>caa1-aDOiCAxFFMOVIX@jetpack</td>
      <td>Movies Start</td>
      <td>0.2.6</td>
      <td>False</td>
      <td>74</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>57</th>
      <td>activations.cdn.mozilla.net^privatebrowsingid=...</td>
      <td>Facebook</td>
      <td>2</td>
      <td>None</td>
      <td>74</td>
      <td>0.01</td>
    </tr>
    <tr>
      <th>58</th>
      <td>{176c8b66-7fc3-4af5-a86b-d0207c456b14}</td>
      <td>Search for Fire Fox</td>
      <td>1.6</td>
      <td>False</td>
      <td>63</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>59</th>
      <td>{58d735b4-9d6c-4e37-b146-7b9f7e79e318}</td>
      <td>Findwide Search Engine</td>
      <td>1.6</td>
      <td>False</td>
      <td>61</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>60</th>
      <td>mozilla_cc2@internetdownloadmanager.com</td>
      <td>IDM integration</td>
      <td>6.26.10</td>
      <td>False</td>
      <td>56</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>61</th>
      <td>firefox@zenmate.com</td>
      <td>ZenMate Security, Privacy &amp; Unblock VPN</td>
      <td>5.9.0</td>
      <td>False</td>
      <td>53</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>62</th>
      <td>@Package</td>
      <td>Package</td>
      <td>0.2.0</td>
      <td>False</td>
      <td>53</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>63</th>
      <td>light_plugin_D772DC8D6FAF43A29B25C4EBAA5AD1DE@...</td>
      <td>Kaspersky Protection</td>
      <td>4.6.2-42-20160922074409</td>
      <td>False</td>
      <td>52</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>64</th>
      <td>@Email</td>
      <td>Email</td>
      <td>4.0.12</td>
      <td>False</td>
      <td>52</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>65</th>
      <td>_fsMembers_@free.pdfconverterhq.com</td>
      <td>PDFConverterHQ</td>
      <td>7.102.10.4849</td>
      <td>False</td>
      <td>51</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>66</th>
      <td>_4zMembers_@www.videodownloadconverter.com</td>
      <td>VideoDownloadConverter</td>
      <td>7.102.10.5033</td>
      <td>False</td>
      <td>49</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>67</th>
      <td>arthurj8283@gmail.com</td>
      <td>xRocket Toolbar</td>
      <td>1.0.1</td>
      <td>False</td>
      <td>48</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>68</th>
      <td>AVJYFVOD75109374@HCDE39471360.com</td>
      <td>CinemaPlus-3.3c</td>
      <td>0.95.114</td>
      <td>False</td>
      <td>47</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>69</th>
      <td>@hoxx-vpn</td>
      <td>Hoxx VPN Proxy</td>
      <td>1.8.9</td>
      <td>False</td>
      <td>46</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>70</th>
      <td>bkavplugin@bkav</td>
      <td>Bkav Plugin</td>
      <td>2.0.2</td>
      <td>False</td>
      <td>45</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>71</th>
      <td>{71A44B6B-42B9-4111-BD15-E67572E92A4C}</td>
      <td>Vision WebLock</td>
      <td>8.6.0</td>
      <td>False</td>
      <td>45</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>72</th>
      <td>eagleget_ffext@eagleget.com</td>
      <td>EagleGet Free Downloader</td>
      <td>4.1.13</td>
      <td>False</td>
      <td>45</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>73</th>
      <td>{1B33E42F-EF14-4cd3-B6DC-174571C4349C}</td>
      <td>Thunder Extension</td>
      <td>4.7</td>
      <td>False</td>
      <td>44</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>74</th>
      <td>{b9acf540-acba-11e1-8ccb-001fd0e08bd4}</td>
      <td>Easy Youtube Video Downloader Express</td>
      <td>9.11</td>
      <td>False</td>
      <td>42</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>75</th>
      <td>{81BF1D23-5F17-408D-AC6B-BD6DF7CAF670}</td>
      <td>iMacros for Firefox</td>
      <td>9.0.3</td>
      <td>False</td>
      <td>42</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>76</th>
      <td>firefox@ghostery.com</td>
      <td>Ghostery</td>
      <td>7.1.1.5</td>
      <td>False</td>
      <td>41</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>77</th>
      <td>_dbMembers_@free.getformsonline.com</td>
      <td>GetFormsOnline</td>
      <td>7.102.10.4251</td>
      <td>False</td>
      <td>40</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>78</th>
      <td>browsec@browsec.com</td>
      <td>Browsec</td>
      <td>2.0.3</td>
      <td>False</td>
      <td>38</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>79</th>
      <td>adbhelper@mozilla.org</td>
      <td>ADB Helper</td>
      <td>0.9.0</td>
      <td>False</td>
      <td>38</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>80</th>
      <td>@DiscreteSearch</td>
      <td>Discrete Search</td>
      <td>0.2.1</td>
      <td>False</td>
      <td>37</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>81</th>
      <td>fxdevtools-adapters@mozilla.org</td>
      <td>Valence</td>
      <td>0.3.5</td>
      <td>False</td>
      <td>37</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>82</th>
      <td>delicious.com@services.mozilla.org</td>
      <td>Delicious</td>
      <td>1.0.0</td>
      <td>None</td>
      <td>37</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>83</th>
      <td>WebProtection@360safe.com</td>
      <td>360 Internet Protection</td>
      <td>5.0.0.1005</td>
      <td>False</td>
      <td>37</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>84</th>
      <td>hotspot-shield@anchorfree.com</td>
      <td>Hotspot Shield Free VPN Proxy – Unblock Sites</td>
      <td>1.2.87</td>
      <td>False</td>
      <td>34</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>85</th>
      <td>_dzMembers_@www.pconverter.com</td>
      <td>PConverter</td>
      <td>7.102.10.4851</td>
      <td>False</td>
      <td>34</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>86</th>
      <td>multifox@hultmann</td>
      <td>Multifox</td>
      <td>3.2.3</td>
      <td>False</td>
      <td>34</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>87</th>
      <td>osb@quicksaver</td>
      <td>OmniSidebar</td>
      <td>1.6.14</td>
      <td>False</td>
      <td>33</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>88</th>
      <td>helper-sig@savefrom.net</td>
      <td>SaveFrom.net - helper</td>
      <td>6.92</td>
      <td>False</td>
      <td>32</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>89</th>
      <td>_euMembers_@free.filesendsuite.com</td>
      <td>FileSendSuite</td>
      <td>7.102.10.4154</td>
      <td>False</td>
      <td>32</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>90</th>
      <td>fdm_ffext@freedownloadmanager.org</td>
      <td>Free Download Manager extension</td>
      <td>2.1.13</td>
      <td>False</td>
      <td>30</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>91</th>
      <td>_gtMembers_@free.gamingwonderland.com</td>
      <td>GamingWonderland</td>
      <td>7.102.10.4263</td>
      <td>False</td>
      <td>29</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>92</th>
      <td>jid1-q4sG8pYhq8KGHs@jetpack</td>
      <td>AdBlocker for YouTube™</td>
      <td>0.2.5</td>
      <td>False</td>
      <td>29</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>93</th>
      <td>87677a2c52b84ad3a151a4a72f5bd3c4@jetpack</td>
      <td>Grammarly for Firefox</td>
      <td>8.698.584</td>
      <td>False</td>
      <td>29</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>94</th>
      <td>safefacebook@bkav</td>
      <td>Bkav SafeFacebook</td>
      <td>1.0.4</td>
      <td>False</td>
      <td>28</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>95</th>
      <td>ar1er-ewrgfdgomusix@jetpack</td>
      <td>Music Start</td>
      <td>1.2.2</td>
      <td>False</td>
      <td>27</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>96</th>
      <td>_agMembers_@free.premierdownloadmanager.com</td>
      <td>PremierDownloadManager</td>
      <td>7.102.10.4846</td>
      <td>False</td>
      <td>27</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>97</th>
      <td>ols@f-secure.com</td>
      <td>Browsing Protection by F-Secure</td>
      <td>2.176.4626</td>
      <td>False</td>
      <td>27</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>98</th>
      <td>{cb40da56-497a-4add-955d-3377cae4c33b}</td>
      <td>McAfee Endpoint Security Web Control</td>
      <td>10.2.0.271</td>
      <td>False</td>
      <td>25</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>99</th>
      <td>dc59fc10-5a26-4311-af8d-bf9b600a7b9c@080e29b9-...</td>
      <td>FLV Player Addon</td>
      <td>0.95.190</td>
      <td>False</td>
      <td>25</td>
      <td>0.00</td>
    </tr>
    <tr>
      <th>100</th>
      <td>jid1-NIfFY2CA8fy1tg@jetpack</td>
      <td>AdBlock for Firefox</td>
      <td>2.5.0</td>
      <td>False</td>
      <td>25</td>
      <td>0.00</td>
    </tr>
  </tbody>
</table>
</div>



### Transform Dataframe to RDD of pings


```python
def row_2_ping(row):
    ping = {
        "payload": {"simpleMeasurements": json.loads(row.simpleMeasurements) if row.simpleMeasurements else {},
                    "histograms": json.loads(row.histograms) if row.histograms else {},
                    "keyedHistograms": json.loads(row.keyedHistograms) if row.keyedHistograms else {},
                    "childPayloads": json.loads(row.childPayloads) if row.childPayloads else {},
                    "threadHangStats": json.loads(row.threadHangStats)} if row.threadHangStats else {},
       "e10s": True if row.e10sCohort.endswith("test") else False,
       "addons": True if row.e10sCohort.startswith("addons") else False,
       "system": json.loads(row.system),
       "cohort": row.e10sCohort
    }
    return ping

def notxp(p):
    os = p.get("system", {}).get("os", {})
    return os["name"] != "Windows_NT" or os["version"] != "5.1"

subset = addons_exp_dataset.rdd.map(row_2_ping).filter(notxp)
```

```python
def add_gecko_activity(ping):
    uptime = ping["payload"].get("simpleMeasurements", {}).get("totalTime", -1) / 60
    if uptime <= 0:
        return ping

    def get_hangs_per_minute(threads, thread_name, uptime):
        for thread in threads:
            if thread["name"] == thread_name:
                activity = thread["activity"]["values"]
                if activity:
                    histogram = pd.Series(activity.values(), index=map(int, activity.keys())).sort_index()
                    # 255 is upper bound for 128-255ms bucket.
                    return histogram[histogram.index >= 255].sum() / uptime
        return None

    threads = ping["payload"].get("threadHangStats", {})
    ping["parent_hangs_per_minute"] = get_hangs_per_minute(threads, "Gecko", uptime)

    child_payloads = ping["payload"].get("childPayloads", [])
    child_hangs_per_minute = []
    for payload in child_payloads:
        child_uptime = payload.get("simpleMeasurements", {}).get("totalTime", -1) / 60
        if child_uptime <= 0:
            continue
        child_threads = payload.get("threadHangStats", {})
        child_hangs = get_hangs_per_minute(child_threads, "Gecko_Child", child_uptime)
        if child_hangs:
            child_hangs_per_minute.append(child_hangs)

    if len(child_hangs_per_minute) > 0:
        ping["child_hangs_per_minute"] = sum(child_hangs_per_minute) / len(child_hangs_per_minute)

    return ping

subset = subset.map(add_gecko_activity)
```
At this point, how many clients are left in each cohort? Key first by cohort.


```python
subset = subset.map(lambda r: (r["cohort"], r))

cohort_sizes = subset.countByKey()
cohort_sizes
```




    defaultdict(int,
                {u'addons-set51alladdons-control': 456484,
                 u'addons-set51alladdons-test': 451602,
                 u'control': 642695,
                 u'test': 625177})



We include the standard e10s cohorts to provide an additional comparison to the addon cohorts. If we see an e10s-related difference for profiles with add-ons, we want to see whether the difference is specific to having add-ons, or whether it occurs regardless.

Since the addon cohorts are much smaller than the standard ones, we draw samples from the standard ones to make them approximately the same size.

NOTE: **MPC=False should be blocked (see [bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1329695)), temporary fix:**
In Beta 51, we have a much larger cohort size for addons since we do not only take MPC=True, so we will sample 25% of the cohort to approximately match previous cohort sizes. This is necessary to continue the analysis due to memory contraints.


```python
target_prop_test = cohort_sizes[ADDONS_TEST_COHORT] / cohort_sizes["test"]
target_prop_control = cohort_sizes[ADDONS_CONTROL_COHORT] / cohort_sizes["control"]
sampling_props = {
    ADDONS_TEST_COHORT: .25,
    ADDONS_CONTROL_COHORT: .25,
    u"test": target_prop_test * .25,
    u"control": target_prop_control * .25    
}
subset = subset.sampleByKey(False, sampling_props)\
    .persist(StorageLevel.MEMORY_AND_DISK_SER)
    
print 'Sampling the following proportions from each group:'
sampling_props

```
    Sampling the following proportions from each group:






    {u'addons-set51alladdons-control': 0.25,
     u'addons-set51alladdons-test': 0.25,
     u'control': 0.1775663417328593,
     u'test': 0.180589657009135}



Now compute the final cohort sizes, and wrap them into the histogram comparison functions.


```python
e10s_addon_cohort_sizes = subset.countByKey()

## Remove the cohort label key from the dataset.
subset = subset.map(lambda r: r[1])
```

```python
print("Final cohort sizes:")
print(" - e10s (with add-ons): {}".format(e10s_addon_cohort_sizes[ADDONS_TEST_COHORT]))
print(" - non-e10s (with add-ons): {}".format(e10s_addon_cohort_sizes[ADDONS_CONTROL_COHORT]))
print(" - e10s (no add-ons): {}".format(e10s_addon_cohort_sizes["test"]))
print(" - non-e10s (no add-ons): {}".format(e10s_addon_cohort_sizes["control"]))

def compare_histograms(pings, *histogram_names, **kwargs):
    return compare_e10s_histograms(pings, e10s_addon_cohort_sizes, *histogram_names, **kwargs)
    
def compare_count_histograms(pings, *histogram_names, **kwargs):
    return compare_e10s_count_histograms(pings, e10s_addon_cohort_sizes, *histogram_names, **kwargs)
```
    Final cohort sizes:
     - e10s (with add-ons): 113145
     - non-e10s (with add-ons): 114208
     - e10s (no add-ons): 113056
     - non-e10s (no add-ons): 113768


## 1.3 Jank


```python
compare_histograms(subset,  
                   "payload/histograms/GC_MAX_PAUSE_MS",
                   "payload/histograms/CYCLE_COLLECTOR_MAX_PAUSE",
                   "payload/histograms/INPUT_EVENT_RESPONSE_MS")
```


Comparison for GC_MAX_PAUSE_MS (with add-ons):


    114163 non-e10s profiles have this histogram.
    113103 e10s profiles have this histogram.
    113101 e10s profiles have the parent histogram.
    102344 e10s profiles have the children histogram.




![png](images/output_41_2.png)


    The probability that the distributions for GC_MAX_PAUSE_MS (e10s merged) (with add-ons) are differing by chance is 0.000.




![png](images/output_41_4.png)


    The probability that the distributions for GC_MAX_PAUSE_MS (parent) (with add-ons) are differing by chance is 0.000.




![png](images/output_41_6.png)


    The probability that the distributions for GC_MAX_PAUSE_MS (children) (with add-ons) are differing by chance is 0.000.




Comparison for CYCLE_COLLECTOR_MAX_PAUSE (with add-ons):


    107879 non-e10s profiles have this histogram.
    106895 e10s profiles have this histogram.
    106878 e10s profiles have the parent histogram.
    100292 e10s profiles have the children histogram.




![png](images/output_41_10.png)


    The probability that the distributions for CYCLE_COLLECTOR_MAX_PAUSE (e10s merged) (with add-ons) are differing by chance is 0.000.




![png](images/output_41_12.png)


    The probability that the distributions for CYCLE_COLLECTOR_MAX_PAUSE (parent) (with add-ons) are differing by chance is 0.000.




![png](images/output_41_14.png)


    The probability that the distributions for CYCLE_COLLECTOR_MAX_PAUSE (children) (with add-ons) are differing by chance is 0.000.




Comparison for INPUT_EVENT_RESPONSE_MS (with add-ons):


    114172 non-e10s profiles have this histogram.
    113126 e10s profiles have this histogram.
    113126 e10s profiles have the parent histogram.
    103127 e10s profiles have the children histogram.




![png](images/output_41_18.png)


    The probability that the distributions for INPUT_EVENT_RESPONSE_MS (e10s merged) (with add-ons) are differing by chance is 0.000.




![png](images/output_41_20.png)


    The probability that the distributions for INPUT_EVENT_RESPONSE_MS (parent) (with add-ons) are differing by chance is 0.000.




![png](images/output_41_22.png)


    The probability that the distributions for INPUT_EVENT_RESPONSE_MS (children) (with add-ons) are differing by chance is 0.000.


## 1.4 Page load


```python
compare_histograms(subset, "payload/histograms/FX_PAGE_LOAD_MS")
```


Comparison for FX_PAGE_LOAD_MS (with add-ons):


    111385 non-e10s profiles have this histogram.
    112687 e10s profiles have this histogram.
    112687 e10s profiles have the parent histogram.
    No e10s profiles have the children histogram.




![png](images/output_43_2.png)


    The probability that the distributions for FX_PAGE_LOAD_MS (parent) (with add-ons) are differing by chance is 0.009.


## 1.5 Startup/shutdown time


```python
simple = pd.DataFrame(get_pings_properties(subset, [
    "payload/simpleMeasurements/firstPaint",
    "payload/simpleMeasurements/sessionRestored",
    "payload/simpleMeasurements/shutdownDuration",
    "e10s",
    "addons"]).collect())

eSimple = simple[simple["addons"] & simple["e10s"]]
nSimple = simple[simple["addons"] & ~simple["e10s"]]
eSimple_std = simple[~simple["addons"] & simple["e10s"]]
nSimple_std = simple[~simple["addons"] & ~simple["e10s"]]

len(eSimple), len(nSimple), len(eSimple_std), len(nSimple_std)
```




    (113145, 114208, 113056, 113768)




```python
compare_scalars("firstPaint time",
                eSimple["payload/simpleMeasurements/firstPaint"],
                nSimple["payload/simpleMeasurements/firstPaint"],
                eSimple_std["payload/simpleMeasurements/firstPaint"],
                nSimple_std["payload/simpleMeasurements/firstPaint"],
                "ms")
```
    Comparison for firstPaint time (ms) (with add-ons):
    
    - Median with e10s is 213 ms different from median without e10s.
    - This is a relative difference of 4.4%.
    - E10s group median is 5053, non-e10s group median is 4840.
    
    The probability of this difference occurring purely by chance is 0.000.
    
    For cohorts with no add-ons, median with e10s is 128 ms (3.1%) different from median without



```python
compare_scalars("sessionRestored time",
                eSimple["payload/simpleMeasurements/sessionRestored"],
                nSimple["payload/simpleMeasurements/sessionRestored"],
                eSimple_std["payload/simpleMeasurements/sessionRestored"],
                nSimple_std["payload/simpleMeasurements/sessionRestored"],
               "ms")
```
    Comparison for sessionRestored time (ms) (with add-ons):
    
    - Median with e10s is -57 ms different from median without e10s.
    - This is a relative difference of -0.9%.
    - E10s group median is 6311, non-e10s group median is 6368.
    
    The probability of this difference occurring purely by chance is 0.088.
    
    For cohorts with no add-ons, median with e10s is -71 ms (-1.3%) different from median without



```python
compare_scalars("shutdownDuration time",
                eSimple["payload/simpleMeasurements/shutdownDuration"],
                nSimple["payload/simpleMeasurements/shutdownDuration"],
                eSimple_std["payload/simpleMeasurements/shutdownDuration"],
                nSimple_std["payload/simpleMeasurements/shutdownDuration"],
               "ms")
```
    Comparison for shutdownDuration time (ms) (with add-ons):
    
    - Median with e10s is 63 ms different from median without e10s.
    - This is a relative difference of 4.5%.
    - E10s group median is 1451, non-e10s group median is 1388.
    
    The probability of this difference occurring purely by chance is 0.000.
    
    For cohorts with no add-ons, median with e10s is 52 ms (4.5%) different from median without


## 1.6 Scrolling


```python
compare_histograms(subset, "payload/histograms/FX_REFRESH_DRIVER_SYNC_SCROLL_FRAME_DELAY_MS")
```


Comparison for FX_REFRESH_DRIVER_SYNC_SCROLL_FRAME_DELAY_MS (with add-ons):


    79981 non-e10s profiles have this histogram.
    34331 e10s profiles have this histogram.
    6136 e10s profiles have the parent histogram.
    31022 e10s profiles have the children histogram.




![png](images/output_50_2.png)


    The probability that the distributions for FX_REFRESH_DRIVER_SYNC_SCROLL_FRAME_DELAY_MS (e10s merged) (with add-ons) are differing by chance is 0.000.




![png](images/output_50_4.png)


    The probability that the distributions for FX_REFRESH_DRIVER_SYNC_SCROLL_FRAME_DELAY_MS (parent) (with add-ons) are differing by chance is 0.000.




![png](images/output_50_6.png)


    The probability that the distributions for FX_REFRESH_DRIVER_SYNC_SCROLL_FRAME_DELAY_MS (children) (with add-ons) are differing by chance is 0.000.


## 1.7 Plugin jank

The plugin jank histograms are keyed by plugin. We find the most common plugin across all three histograms, and make the comparisons for that plugin.


```python
plugin_hist = ["BLOCKED_ON_PLUGIN_MODULE_INIT_MS",
               "BLOCKED_ON_PLUGIN_INSTANCE_INIT_MS",
               "BLOCKED_ON_PLUGIN_INSTANCE_DESTROY_MS"]

def get_hist_plugins(ping):
    """ Find the keys used across all plugin histograms. """
    khist = ping.get("payload", {}).get("keyedHistograms", {})
    plugin_keys = []
    for h in plugin_hist:
        if h in khist:
            plugin_keys += map(lambda k: (h, k), khist[h].keys())
    return plugin_keys
        
plugin_hist_counts = subset.flatMap(get_hist_plugins).countByValue()
## Find the most commonly occurring plugin for each histogram.
top_plugins = {}
for h in plugin_hist:
    pl_for_hist = [(pl, n) for ((hist, pl), n) in plugin_hist_counts.iteritems()
                       if hist == h]
    top_plugins[h] = sorted(pl_for_hist, key=lambda (pl, n): n, reverse=True)[0]

for hist, (pl, n) in top_plugins.iteritems():
    print("Top plugin for {}: '{}'".format(hist, pl))
```
    Top plugin for BLOCKED_ON_PLUGIN_MODULE_INIT_MS: 'Shockwave Flash24.0.0.186'
    Top plugin for BLOCKED_ON_PLUGIN_INSTANCE_DESTROY_MS: 'Shockwave Flash24.0.0.186'
    Top plugin for BLOCKED_ON_PLUGIN_INSTANCE_INIT_MS: 'Shockwave Flash24.0.0.186'



```python
top_plugin = sorted(top_plugins.items(), key=lambda (pl, n): n, reverse=True)[0]
top_plugin = top_plugin[1][0]
print("Comparing plugin jank for '{}' (overall top plugin)".format(top_plugin))
```
    Comparing plugin jank for 'Shockwave Flash24.0.0.186' (overall top plugin)



```python
compare_histograms(subset,
                   "payload/keyedHistograms/BLOCKED_ON_PLUGIN_MODULE_INIT_MS/{}".format(top_plugin),
                   "payload/keyedHistograms/BLOCKED_ON_PLUGIN_INSTANCE_INIT_MS/{}".format(top_plugin),
                   "payload/keyedHistograms/BLOCKED_ON_PLUGIN_INSTANCE_DESTROY_MS/{}".format(top_plugin))
```


Comparison for [BLOCKED_ON_PLUGIN_MODULE_INIT_MS](https://dxr.mozilla.org/mozilla-central/search?q=BLOCKED_ON_PLUGIN_MODULE_INIT_MS+file%3AHistograms.json&redirect=true)/Shockwave Flash24.0.0.186 (with add-ons):


    12028 non-e10s profiles have this histogram.
    11077 e10s profiles have this histogram.
    11077 e10s profiles have the parent histogram.
    10356 e10s profiles have the children histogram.




![png](images/output_55_2.png)


    The probability that the distributions for BLOCKED_ON_PLUGIN_MODULE_INIT_MS (e10s merged) (with add-ons) are differing by chance is 0.009.




![png](images/output_55_4.png)


    The probability that the distributions for BLOCKED_ON_PLUGIN_MODULE_INIT_MS (parent) (with add-ons) are differing by chance is 0.000.




![png](images/output_55_6.png)


    The probability that the distributions for BLOCKED_ON_PLUGIN_MODULE_INIT_MS (children) (with add-ons) are differing by chance is 0.009.




Comparison for [BLOCKED_ON_PLUGIN_INSTANCE_INIT_MS](https://dxr.mozilla.org/mozilla-central/search?q=BLOCKED_ON_PLUGIN_INSTANCE_INIT_MS+file%3AHistograms.json&redirect=true)/Shockwave Flash24.0.0.186 (with add-ons):


    12028 non-e10s profiles have this histogram.
    10356 e10s profiles have this histogram.
    No e10s profiles have the parent histogram.
    10356 e10s profiles have the children histogram.




![png](images/output_55_10.png)


    The probability that the distributions for BLOCKED_ON_PLUGIN_INSTANCE_INIT_MS (children) (with add-ons) are differing by chance is 0.955.




Comparison for [BLOCKED_ON_PLUGIN_INSTANCE_DESTROY_MS](https://dxr.mozilla.org/mozilla-central/search?q=BLOCKED_ON_PLUGIN_INSTANCE_DESTROY_MS+file%3AHistograms.json&redirect=true)/Shockwave Flash24.0.0.186 (with add-ons):


    11942 non-e10s profiles have this histogram.
    10188 e10s profiles have this histogram.
    No e10s profiles have the parent histogram.
    10188 e10s profiles have the children histogram.




![png](images/output_55_14.png)


    The probability that the distributions for BLOCKED_ON_PLUGIN_INSTANCE_DESTROY_MS (children) (with add-ons) are differing by chance is 0.598.


## 1.8 Memory usage


```python
compare_histograms(subset,
                   "payload/histograms/MEMORY_TOTAL",
                   "payload/histograms/MEMORY_VSIZE_MAX_CONTIGUOUS")
```


Comparison for MEMORY_TOTAL (with add-ons):


    114150 non-e10s profiles have this histogram.
    112394 e10s profiles have this histogram.
    112394 e10s profiles have the parent histogram.
    No e10s profiles have the children histogram.




![png](images/output_57_2.png)


    The probability that the distributions for MEMORY_TOTAL (parent) (with add-ons) are differing by chance is 0.000.




Comparison for MEMORY_VSIZE_MAX_CONTIGUOUS (with add-ons):


    111961 non-e10s profiles have this histogram.
    110810 e10s profiles have this histogram.
    110802 e10s profiles have the parent histogram.
    91432 e10s profiles have the children histogram.




![png](images/output_57_6.png)


    The probability that the distributions for MEMORY_VSIZE_MAX_CONTIGUOUS (e10s merged) (with add-ons) are differing by chance is 0.000.




![png](images/output_57_8.png)


    The probability that the distributions for MEMORY_VSIZE_MAX_CONTIGUOUS (parent) (with add-ons) are differing by chance is 0.000.




![png](images/output_57_10.png)


    The probability that the distributions for MEMORY_VSIZE_MAX_CONTIGUOUS (children) (with add-ons) are differing by chance is 0.000.


## 1.9 UI Smoothness

__Note__: `FX_TAB_SWITCH_TOTAL_MS` was renamed to `FX_TAB_SWITCH_TOTAL_E10S_MS` for e10s profiles.


```python
def fix_hist(ping):
    """ Rename the histogram for e10s profiles. """
    hist = ping.get("payload", {}).get("histograms", {})
    if "FX_TAB_SWITCH_TOTAL_E10S_MS" in hist and "FX_TAB_SWITCH_TOTAL_MS" not in hist:
        hist["FX_TAB_SWITCH_TOTAL_MS"] = hist["FX_TAB_SWITCH_TOTAL_E10S_MS"]
    return ping

subset_fixed = subset.map(fix_hist)
```

```python
compare_histograms(subset_fixed, "payload/histograms/FX_TAB_SWITCH_TOTAL_MS")
```


Comparison for FX_TAB_SWITCH_TOTAL_MS (with add-ons):


    72026 non-e10s profiles have this histogram.
    70295 e10s profiles have this histogram.
    70295 e10s profiles have the parent histogram.
    No e10s profiles have the children histogram.




![png](images/output_61_2.png)


    The probability that the distributions for FX_TAB_SWITCH_TOTAL_MS (parent) (with add-ons) are differing by chance is 0.000.


## 1.11 Slow Scripts


```python
compare_count_histograms(subset, "payload/histograms/SLOW_SCRIPT_PAGE_COUNT")
```


Comparison for count histogram SLOW_SCRIPT_PAGE_COUNT (with add-ons):


    2294 non-e10s profiles have this histogram.
    2854 e10s profiles have this histogram.
    
    Comparison for SLOW_SCRIPT_PAGE_COUNT per hour (with add-ons):
    
    - Median with e10s is 0.0516 units different from median without e10s.
    - This is a relative difference of 16.6%.
    - E10s group median is 0.3629, non-e10s group median is 0.3113.
    
    The probability of this difference occurring purely by chance is 0.018.
    
    For cohorts with no add-ons, median with e10s is -0.00771 units (-2.4%) different from median without

At Proagrica we’ve been using HPCC for our data storage and transformation for a couple of years. We’ve learnt a lot over this 
time but there’s something that comes up quite frequently while we are testing, doing quick investigations or making some model 
features. The issue is that ECL can be a bit verbose and sometimes simple things are not simple to remember. It can be especially 
challenging among occasional users who may need to lookup things whenever they open the IDE.  

We felt this pain so developed n ECL bundle to make coding faster, more logical and, most of all, easier to remember. This package 
has become known as dapper and it allows us to quickly subset, transform and summarise data. If you’ve used R’s dplyr package you 
may well get what we mean!  

Intrigued? Me too. Let’s look at what it can do! 

# Setup

## Bundles 

Dapper is available as an ECL bundle. Something that we only recently learned is that HPCC actually supports libraries in a 
similar way to Python's pip. It isn't quite as fully featured but it works well enough. The basic idea is that it will 
pull down a 'bundle' of ECL code from github and install it locally *making it available as* 
*if it was part of the core libraries*. That is, you can `IMPORT` it without having to have the scripts in your repo. 

This tool is made available with ecl.exe (and its Linux equivalent). To install the latest dapper release you simply run the 
following on the command line (although I hear that the latest IDE has this baked in).

```sh
ecl bundle install -v https://github.com/OdinProAgrica/dapper.git
```

There's many more bundles available, but their coverage, testing and adherence to version control varies. HPCC keep a curated 
list [here](https://github.com/hpcc-systems/ecl-bundles).

## IMPORTing dapper

**Do note that the standard nomenclature for importing the modules should be respected.** This is because each module references 
its own functions, requiring a known import name. Modules should always be imported as:

* TransformTools: tt  
* StringTools: st  

For example: 
```ECL
IMPORT dapper.TransformTools as tt;
IMPORT dapper.StringTools as st;
```

**inDS and OutDS are reserved** You may find you get weird errors if you use these variable names, this will be fixed in a later 
release.


# So, what can it do? 

Once installed, dapper can be used to create scripts using simple verbs which can increase readability and decrease coding 
mistakes. In short, it reduces the amount of time you spend thinking about *how* to do a job, you can just get on with it. Don’t 
get me wrong, sometimes an old school `PROJECT` is better. I leave it to the reader to decide what they like. As always, 
right tools, right job. It is worth noting however that, even if you use several transformTools statements in a row, the 
compiler is clever enough to combine this into a single operation under the bonnet, minimising dapper's speed impact. 

The bundle itself is broken down into two sets of tools. This post will focus on our transform tools. I 
will do a separate post on stringtools (and our in development bundle geodapper) in the future watch this space!


# Transform Tools

Have a quick skim over dappper's key functions below, I've just chosen a subset here, a more complete list is available
on the [project's github](https://github.com/OdinProAgrica/dapper) and includes things like filter operations. Once 
you've got the gist I'll give you an ECL script we can pick apart. 

### Data Transformations
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/DataTransformations.PNG)

### Duplicates
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/DupsDedups.PNG)

### Column Transforms
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/Columns.PNG)

### Filters
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/Filters.PNG)

### Arrangement
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/Arrange.PNG)

### Outputs
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/Outputs.PNG)

### Summaries
![](https://github.com/odinproagrica/DocumentationImages/blob/master/TransformTools/Summaries.PNG)


# How does it work? 

Okay, so I'm going to give you an ECL script which (assuming you have installed dapper using the bundle method described above) 
will work on your system (fingers crossed!). 

```ECL
IMPORT dapper.ExampleData;
IMPORT dapper.TransformTools as tt;

//load data
StarWars := ExampleData.starwars;


// Look at the data
tt.nrows(StarWars);
tt.head(StarWars);


//Fill blank species with unknown
fillblankHome := tt.mutate(StarWars, species, IF(species = '', 'Unkn.', species));
tt.head(fillblankHome);

//Create a BMI for each character
bmi := tt.append(fillblankHome, REAL, BMI, mass/height^2);
tt.head(bmi);


//Find the highest
sortedBMI := tt.arrange(bmi, '-bmi');
tt.head(sortedBMI);
//Jabba should probably go on a diet. 


//How many of each species are there?
species := tt.countn(sortedBMI, 'species');
sortedspecies := tt.arrange(species, '-n');
tt.head(sortedspecies);


//Finally let's look at unique hair/eye colour combinations:
colourData := tt.select(StarWars, 'hair_color, eye_color'); 
unqiueColours := tt.distinct(colourData, 'hair_color, eye_color'); //see arrangedistinct() for fancy sort/dedup operations. 
tt.head(unqiueColours);
tt.countn(unqiueColours);


//and save our results
tt.to_csv(sortedBMI, 'ROB::TEMP::STARWARSCSV');
```


## So let's break some of this down. 


### Example Dataset
```ECL
IMPORT dapper.ExampleData;
IMPORT dapper.TransformTools as tt;

//load data
StarWars := ExampleData.starwars;
```

Yes, I'm a nerd, what of it? 

### Viewing Data

```ECL
// Look at the data
tt.nrows(StarWars);
tt.head(StarWars);
```

These are shorthand for `OUTPUT` and `COUNT` however note the way the results are named. It automagically renames your results 
to match your variables. 


### Transformations

```ECL
//Fill blank species with unknown
fillblankHome := tt.mutate(StarWars, species, IF(species = '', 'Unkn.', species));
tt.head(fillblankHome);

//Create a BMI for each character
bmi := tt.append(fillblankHome, REAL, BMI, mass/height^2);
tt.head(bmi);

//Find the highest
sortedBMI := tt.arrange(bmi, '-bmi');
tt.head(sortedBMI);
//Jabba should probably go on a diet. 
```

Mutate and append allow column transforms via a simple formula. Note that there is no need for `SELF` or `LEFT` in these 
transforms, making them easier to write and easier to read!

### Grouped Counts

```ECL
//How many of each species are there?
species := tt.countn(sortedBMI, 'species');
sortedspecies := tt.arrange(species, '-n');
tt.head(sortedspecies);
```

The record definition to do a cross-tab is something I always have to look up. `countn` will do it for you, you can 
even hand multiple columns to it and it'll handle them all perfectly.

### Deduplication and Column Selection
```ECL
//Finally let's look at unique hair/eye colour combinations:
colourData := tt.select(StarWars, 'hair_color, eye_color'); 
unqiueColours := tt.distinct(colourData, 'hair_color, eye_color'); 
tt.head(unqiueColours);
tt.countn(unqiueColours);
```

`select` (and it's partner function `drop`) will help in quickly sub-setting data, note too the use of distinct which is 
shorthand for `DEDUP(SORT(DISTRIBUTE(...), LOCAL), LOCAL)`, using `arrangedistinct()` instead allows you to control the sort 
and distribute commands separately. See also `duplicated` which will flag all duplicates with a boolean for investigation!

Also, sorry for the UK English, sometimes I can't help myself!

### Saving Results
```ECL
//and save our results
tt.to_csv(sortedBMI, 'ROB::TEMP::STARWARSCSV');
```

Finally we can write out. `OUTPUT(...CSV(...))` is another one of those functions you always forget, this handles all the quote, 
separator, header stuff for you and simply writes out a 'normal' csv file. It'll also add the tilde (~) to the start
of your file name if you forget. 

# Summary

That's a quick whistle stop tour of dapper's power and functionality. I hope you can see how useful it is for things like testing
and investigative work, allowing a more logical flow and readable code. If you do have any comments or suggestions please do 
checkout the github issues page [here](https://github.com/OdinProAgrica/dapper/issues). Contributions also welcome. 
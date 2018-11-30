ADD LOGO


# Installation

First off, we need to install dapper. You can download it from the github and place it in a repo manually but ECL also allows you to install bundles, 
making the function available as an import automatically. In a terminal (or PowerShell), type the following:

```ECL
ecl bundle install -v https://github.com/OdinProAgrica/dapper.git
```

You can now import dapper's tools as you would any pre-installed package, such as STD:

```ECL
IMPORT dapper.transformtools AS tt;
IMPORT dapper.stringtools AS st;
```

Note the use of `AS tt` and `AS st` for the two parts of the bundle. You must always import with these shortcuts to prevent 
errors in the underlying code. 

# Reading in and exploring

For our example data, I'm using a set of CSVs from a library database I auto generated a while back (I know, I need a hobby). They are purposefully 
messy and available here [LINK]. We will explore, tidy and join these.  

You'll need to spray them up manually (or you could use one of our other packages, hpycc [LINK!!]!) and save them under the same file names if you 
want to copy and paste this code. 

```ECL
bookRec := {STRING UID; STRING author; INTEGER length; INTEGER publication_date; INTEGER rental_period; STRING subject; STRING title;};
books := DATASET('~TEMP::BOOKS', bookRec, THOR);
borrowerRec := {INTEGER DoB; STRING UID; STRING address; STRING ethnic_group; STRING gender; STRING join_date;};
borrowers := DATASET('~TEMP::borrowers', borrowerRec, THOR);
rentalsRec := {STRING UID; STRING book; STRING borrower, INTEGER out, INTEGER returned};
rentals := DATASET('~TEMP::rentals', rentalsRec, THOR);
```

Lovely. Now if you are anything like me, the first thing you'll want to do is take a look at your shiny new tables. Here we can see the first couple 
of transformtools functions, which print the top 100 rows (by default) and produce a count. 

```ECL
tt.head(books);
tt.nrows(books);
tt.head(borrowers);
tt.nrows(borrowers);
tt.head(rentals);
tt.nrows(rentals);
```

I know what you are thinking: "That's just a wrapper for `OUTPUT()` and `OUTPUT(LEN())`. Yes...but more than that. Run it and come back to me. 
Did you notice the names of the results?! This provides a shorthand for `OUTPUT(..., NAMED())` which is a bit verbose for my liking.   

# Data Summaries
So, we now have our data, what do we want to do now? Well, a quick glance suggests that the books.subject column is a tad.....off. Let's have a 
look at what it contains. I've asked `tt.head()` to give me more data this time (500 rows) so I can get a better idea of what's there.

```ECL
BookSubjectCount := tt.countn(books,  'subject');
tt.head(BookSubjectCount, 500);
```

Some serious data entry issues here! Could do with knowing what the most frequent are though:

```ECL
TopBookSubjectCount := tt.arrange(BookSubjectCount, 'n');
tt.head(TopBookSubjectCount);
```

At least we now know that SOMETHING IS TRUE. I hope you are starting to pick up a little bit of the syntax here, for the most part your datasets 
are given as code, column names are passed as a string. This allows us to pass multiple columns if needed. For example if I wanted a count of book 
rental period (days allowed out) by subject:

```ECL
BookPeriodSubjectCount := tt.nrows(books, 'rental_period, subject');
tt.head(BookPeriodSubjectCount, 500);
```

We will return to tidying up this chaos later!

# Data Manipulation???
For now, let's join our data together a little to find out who has borrowed what and when. 
Take another look at our three tables, you'll first of all note that some fields have names that may be confusing after joining. For example, 
book contains a field called length. This is probably number of pages but once it's joined to rentals it could mean rental length too! Let's 
rename it to prevent confusion.

```ECL
fixLength := tt.rename(books, length, num_pages);
Easy! I'm also going to do this on the UID fields in borrower and book to make my joins easier:
fixBookUID := tt.rename(fixLength, uid, book_uid);
fixBorrowerUID := tt.rename(borrower, uid, borrower_uid);
```

I can already hear what you are saying: “The last three calls could have been one project!” You are of course right and if you have a lot of these I would use that, although we are working on ideas to allow these functions to be more seamlessly combined. The key difference is that there is a lot of verbosity associated with a PROJECT call and the compiler is clever enough to see that what you want is one operation so it helpfully combines the calls. There is therefore no compute cost to this layout! 
Let's join our tables, no fancy functions for this I'm afraid, not yet anyway! 

```ECL
bookToRental := JOIN(rentals, fixBookUID, INNER, SMART);
rentalsBooksBorrowers := JOIN(fixBorrowerUID, bookToRental, INNER, SMART);
tt.head(rentalsBooksBorrowers);
```

Very few joins are perfect so let’s check if all the borrowers have been joined to rentals. There’s plenty of ways to do this in base ECL but none are quick to write. Instead, let’s use another dapper function:
BorrowerList := tt.distinct(rentalsBooksBorrowers, 'borrower_uid');

```ECL
tt.nrows(BorrowerList); 
```

Note that the borrower table and this new list are both the same length. Easy, no?   

There's more cool stuff you can do here. Let's say you want the first book borrowed by each borrower (I don't know about you but I'm curious). We 
can do a distinct but we need first to sort and distribute the data properly. Fear not, dapper can still help here!
```ECL
FirstBook := tt.arrangedistinct(rentalsBooksBorrowers, 'borrower_uid', 'borrower_uid, out', 'borrower_uid');
tt.head(FirstBook);
```

 This is a shorthand for `DEDUP(SORT(DISTRIBUTE(), LOCAL), LOCAL)` The arguments given are exactly that. I am deduping on author, sorting by author 
 and earliest date and then deduping on author. 

The thing with distinct is, it loses you data. If you want to look at the reasons for duplication you have to work a bit harder. I'm tired just 
thinking about how to filter by duplicates in ECL! 

If I wanted to look at duplicated rental events (based on duplicated IDs) YOU NEED THIS IN YOUR DATA!!!! I could use:

```ECL
FindDuplicatedRentals := tt.duplicated(rentalsBooksBorrowers, 'rental_uid');  PROBABLY NEED TO RENAME THIS EARLIER!!!!!
duplicatedRentals := FindDuplicatedRentals(duplicated_rental_uid);  //You could use NOT here to find uniques too!
tt.head(duplicatedRentals);
```

If they are there there's a serious error occurring but what is it? using `tt.duplicated()` you will add a boolean column to your data in the form of 
duplicated_\[column name\], indicating if it's a duplicated value or not. All duplicates will be flagged, not just those after the first, making it great 
for investigation.  

Looks like something is up but I need to know all the duplicated records are next to each other for comparison.

```ECL
sortedDuplicatedRentals := tt.arrange(duplicatedRentals, 'rental_uid');
tt.head(sortedDuplicatedRentals, 1000);
```

Better! Looks like whenever we see duplicates they are exactly that, duplicates. We can thus safely dedup the whole dataset on rental_uid but to be safe I'm also running on book_uid and borrower_uid!

```ECL
uniqueRentals := tt.distinct(rentalsBooksBorrowers, 'rental_uid', 'borrower_uid', 'book_uid');
tt.nrow(uniqueRentals);
```

At this point I'd validate this by doing some deduping in rentals and checking but run with me for now. We've seriously reduced the size of our data 
and got rid of potential bias in what we are looking at! 

So let's say we want a new column, isLate. This will tell us if a particular rental resulted in a late return. No need for a PROJECT here, I can just 
crack on with some dapper tools. Note that I don't need LEFT here for any column names, it can work that out itself. I simply say 
`tt.append([dataset], [new column type], [new column name], [transform statement]).` For example:

```ECL
IMPORT std
daysOut := tt.append(uniqueRentals, INTEGER, DaysOut, STD.Date.DaysBetween(out, in));
isLate := tt.append(daysOut, BOOLEAN, isLate, DaysOut > rental_period);
tt.head(isLate);
In fact I don't really need daysOut though so let's drop it. I could have just done the transform in one go but then I wouldn't be able to show you this:
DropDaysOut := tt.drop(isLate, 'daysOut');
tt.head(DropDaysOut);
```

You could also use `tt.select()` if you only wanted a few columns, pick whichever is easier. Both will ake as many columns as you want to add. 
One job left. Those accursed subject names. We should have fixed them before the JOIN but we weren't quite ready for those functions yet. Let's say we 
want to lower case them first:

```ECL
lowerCaseSubject := tt.mutate(DropDaysOut, subject, STD.Str.ToLowerCase(subject);
lowerCaseSubjectCount := tt.countn(lowerCaseSubject, 'subject');
tt.head(lowerCaseSubjectCount, 500);
```

Better but we aren't out of the woods yet! The classical fix for this now would be to throw loads of REGEX at it to gradually tidy it up, one 
statement at a time. This is very, very verbose and makes huge projects (or mutates!) with lots of variable names. Stringtools provides a better 
way:

```ECL
RegexList := DATASET([
    {'-'                , ' '},
    {'science fiction'  , 'sci fi'}, 
    {'scifi'            , 'sci fi'},
//    .....ADD THE REST
], {STRING Regex; STRING repl}); note that this is importable from string tools!
```

Giving this to string tools, it will spin through each regex, apply it to your data and cough out the end result. You code is easier to write, cleaner and, most of all, easier to read, debug and modify later. 

```ECL
tidiedSubject := tt.mutate(lowerCaseSubjectCount, subject, regexLoop(subject, RegexList));
tidiedSubjectCount := tt.countn(tidiedSubject, 'subject');
tt.head(tidiedSubjectCount, 500);
```

I don't know about you but I enjoyed that. 

So, we are nearing the end of this introduction to our new toolset. I hope you can see how this makes an analytical workflow much easier as you have 
fewer keystrokes and much less time going "So how do I do that again?". There will be a more detailed post on string tools in the future but for now, 
I want to just end on the three transformtools functions that I have thus far neglected. 

The first two are actually of use here. I have now created a 'clean' dataset that we can use for future investigation and modelling work (I haven't, 
it's still full of crap but we've already suspended our disbelief in this regard haven't we?). Now I want to save it! First as a THOR file for use on 
HPCC and second as a CSV for despraying (although again I point you to hpycc [LINK] for despraying THOR files, it's awesome!). 

THOR files are easy either way, my way is simply a little less verbose:

```ECL
tt.to_thor(tidiedSubject, '~library::tidiedData'); 
```

Can you remember how to do an OUTPUT with a properly formatted CSV? No, me neither. This one uses the 'standard' delimiter (a comma) and the 
'standard' quote character (double quotes).

```ECL 
tt.to_csv(tidiedSubject, 'library::tidiedDataCsv');
```

I suppose you noticed I forgot my tilde (~) there? Do not fret, as I have never seen a use case where you *want* it to default to `thor::filename`, 
this function detects it's missing and adds it in. You're welcome. 

...and finally:

```ECL
tt.filter()
``` 

Will filter one dataset based on another, yes you can write a join for that but this is easier and it optimises your join by first 
deduping and then dropping superfluous columns from the set you are filtering by. This simply didn't fit in this narrative. My bad. You can see 
details of this, and all of dappers functions on the bundle's github: HERE. 



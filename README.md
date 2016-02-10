### Credits ###

This tool was inspired by the work done by [Ryan LaNeve](https://github.com/rlaneve) in his https://github.com/rlaneve/accurev2git repository and the desire to improve it. Since this script is sufficiently different I have placed it in a separate repository here. I must also thank [Tom Isaacson](https://github.com/parsley72) for his contribusion to the discussions about the tool and how it could be improved. It was his work that prompted me to start on this implementation. You can find his fork of the original repo here https://github.com/parsley72/accurev2git.

The algorithm used here was devised by [Robert Smithson](https://github.com/fatfreddie) whose stated goal is to rid the multiverse of AccuRev since ridding just our verse is not good enough.

My work is merely in the implementation and I humbly offer it to anyone who doesn't want to remain stuck with AccuRev.


### Overview ###

AccuRev2Git is a tool to convert an AccuRev depot into a git repo. A specified AccuRev stream will be the target of the conversion, and all promotes to that stream will be turned into commits within the new git repository.

### Getting started ###
- Install python 3.4

- Make sure the paths to the `accurev` and `git` executables are correct for your machine, and that git default configuration has been set.

- Clone the **ac2git** repo.

- Run `python ac2git.py --help` to see all the options. _It is recommended that you at least take a look._

- Run `python ac2git.py --example-config` to get an example configuration. _It is recommended that you at least take a look._

- Follow the steps outlined in the **How to use** section.

### Tested with ###
- `AccuRev 6.1.1 (2014/05/05)`, `git version 2.1.0` and `Python 2.7.8` on a Fedora 21 host.

- `AccuRev 6.1.1 (2014/05/05)`, `git version 1.9.0.msysgit.0` and `Python 2.7.6` on a Window 7 host.

- `Accurev 6.0`, `git 1.9.5` on a Windows 8.1 host. By [Gary](https://github.com/bigminer) in [this comment](https://github.com/orao/ac2git/issues/13#issuecomment-136392393) from issue #13.

### How to use ###

#### Converting a Depot's Streams ####
- Make an example config file:

 ```
 python ac2git.py --example-config
 ```

- Modify the generated file, whose filename defaults to `ac2git.config.example.xml`, (there are plenty of notes in the file and the script help menu to do this)

- Rename the `ac2git.config.example.xml` file as `ac2git.config.xml`

- Modify the configuration file and add the following information:

 - AccuRev username & password
 
 - The name of the Depot. You may only convert a single depot at a time and it is recommended that one Depot is mapped to one git repository.

 - The start & end transactions which correspond to what you would enter in the `accurev hist` command as the `<time-spec>` (so a number, the keyword `highest` or the keyword `now`).

 - The list of streams you wish to convert (must exist in the Depot).

 - The path to the git repository that the script is to create. The folder must exist and should preferably be empty, although it is ok for it to be an existing git repository.

 - A user mapping from AccuRev usernames to git. _Hint: Run `accurev show -fi users` to see a list of all the users which you might need to add._

 - Choose the preferred method to use for converting the streams. If the streams are sparse (the transactions that have changed the stream contents are far apart) I recommend the `deep-hist` method, otherwise the `diff` method may be more optimal (read the _"Converting the contents of a single stream"_ section for more details). If in doubt, use `deep-hist`.

- Run the script

 ```
 python ac2git.py
 ```

- If you encounter any trouble. Run the script with the `--help` flag for more options.

 ```
 python ac2git.py --help
 ```

### The result ###

What this script will spit out is a git repository with independent orphaned branches representing your streams. Meaning, that each stream is converted separately on a branch that has no merge points with any other branch.
This is by design as it was a simpler model to begin with.

Each git branch accurately depicts the stream from which it was created w.r.t. **time**. This means that at each point in time the git branch represents the state of your stream. Not only are the transactions for this stream commited to git but so are any transactions that occurred in the parent stream which automatically flowed down to us.
When combined with my statement from the previous paragraph, this implies that you will see a number of commits on different branches with the same time, author and commit message, most often because they represent the same _promote_ transaction.

Ideally, if you have promoted all of your changes to the parent stream this should be identified as a merge commit and recorded as such. Though it would now be possible to extend this script to do so, it is not on my radar for now as it would be a reasonably large undertaking.
However, there is hope because I've implemented an experimental feature, described below, that does just that but it operates as a post processing step. It is still a little buggy and requires iteration but it proves the concept. Patches are welcomed!

### How it works ###

The conversion is split into two stages. The first stage retrieves the information from Accurev that is needed to construct a Git history while the second stage processes the retrieved information and constructs Git branches for your streams from it.

This method was developed in colaboration with [Robert Smithson](https://github.com/fatfreddie).

#### First stage: retrieval ####

The first stage is controlled by the `--method` argument to the script or the `<method>` tag in the script config file. Depending on the method flag, further described in the _"Converting the contents of a single stream"_ section below, this stage can be quick or slow. For all cases this stage only stores the transactions which affect the streams that were specified in the config file, or all streams if no streams were specified.

The downloaded information is stored in Git, and is stored in the refs found in the `.git/refs/ac2git/` folder of the repository. I will refer to these refs as being in the _"ac2git namespace"_ and will ommit the `.git/` part of their path in further text.

For each transaction that affects the stream that we are processing we retrieve 4 pieces of information from Accurev:
 - The output of the `accurev hist -p <depot> -t <transaction_number> -fex` command, stored in the `hist.xml` file.
 - The output of the `accurev show streams -t <transaction_number> -fixg` command, stored in the `streams.xml` file.
 - The output of the `accurev diff -a -i -v <stream_name> -V <stream_name> -t <transaction_number - 1>-<transaction_number>` command, stored in the `diff.xml` file. This file is missing for `mkstream` transactions.
 - The result of the `accurev pop -R -v <stream_name> -L <git_repo_dir> -t <transaction_number>` command.

_Note: The TaskId field in each accurev command's XML output is set to zero in order to optimize the use of git's hashing of blobs (i.e. the same command output will always hash to the same blob if we replace this one field, saving some minimal space and processing on the git side and making diffs more useful between different commits)._

The first 3 items, `hist.xml`, `streams.xml` and `diff.xml` are committed together, for each transaction, on the `ref/ac2git/<depot_name>/streams/stream_<stream_number>_info` ref with the commit message that has the following format `transaction <transaction_number>`. This is meta-data needed to make decisions about the stream w.r.t. other streams later on and is used to produce merges when processing git branches in the second stage.

The 4th item is the actual state of the stream at this transaction and will be the contents of the Git commit for this transaction while the message will come from the `hist.xml` mentioned earlier. The contents is committed on a separate ref, `ref/ac2git/<depot_name>/streams/stream_<stream_number>_data`, with the commit message also formatted as `transaction <transaction_number>`.

You can inspect these 'hidden branches' with `git log ref/ac2git/<depot_name>/streams/stream_<stream_number>_data` to see their contents.

To view the output of any individual file you can use `git show <commit_hash>:<filename>` so viewing what the last transaction's `hist.xml` looks like you can run the following command `git show ref/ac2git/<depot_name>/streams/stream_<stream_number>_info` where `<depot_name>` and `<stream_number>` are placeholders that you'll need to replace with information from your Accurev depot.

Since the contents of the two refs are so different it would be inefficient to do a `git checkout` of one and then the other for each transaction so the script first processes all the transactions for the `ref/ac2git/<depot_name>/streams/stream_<stream_number>_info` and then checks out the `ref/ac2git/<depot_name>/streams/stream_<stream_number>_data` ref and processes the same transactions it just added to the `..._info` ref. Even if there are no changes the `..._data` ref is always committed to (`--allow-empty` is set for the `git commit`) so that each transaction on the `..._info` ref has a corresponding transaction on the `..._data` ref.

#### Second stage: processing ####

Work in progress...

### Converting the contents of a single stream ###

There are three methods available for converting an accurev stream into a single, orphaned, git branch. The resulting git branch's commits represent transactions that have changed to _contents_ of the stream at various points in time. Each method described is an optimization of the previous and will run quicker but may not be possible to use on an older version of accurev.

The method can be specified in the config file and is documented in the example config with the `<method>` tag (see `python ac2git.py --help` for the `--example-config` option), or specified on the command line by passing the `--method` option. See `python ac2git.py --help` for details.

All methods begin by finding the `mkstream` transaction for each stream and populating it into a fresh branch. All methods create a hidden orphaned git branch (in the `.git/refs/ac2git/` folder) for each indivitual stream.

#### Pop method (slow) ####

The first method is the one Ryan LaNeve implemented, which I call the _pop method_, which works like this:
 - Find the `mkstream` transaction and populate it.
 - Populate it in full and commit into git as an orphaned branch.
 - Start loop:
  + Increment the transaction number by 1
  + Delete the contents of the git repository.
  + Populate the transaction and commit it into git.
  + Repeat loop until done.

This method is really slow since you're always getting the whole history from accurev at each point in time. It does have one optimization that only works for the root stream (which has no parents by definition), in that getting the history for just that stream is sufficient to give you a smaller and still accurate list of transactions to process.

_Ryan only processed the root stream in his script._

#### Diff method ####

The second and third method were devised by [Robert Smithson](https://github.com/fatfreddie) and are a lot faster than the _pop method_ but rely on some features that came in the AccuRev 6.1 client.

I refer to the second method as the _diff method_ and it is a simple optimisation over the _pop method_. It works as follows:
 - Find the `mkstream` transaction and populate it.
 - Populate it in full and commit into git as an orphaned branch.
 - Start loop:
  + Increment the transaction number by 1
  + _Do an_ `accurev diff -a -i -v <stream> -V <stream>` _between this transaction and the last transaction that we populated._
  + _Delete only the files that_ `accurev diff` _reported as changed from the git repository._
  + Populate the transaction and commit it into git. _(The populate here is done with the recursive option but without the overwrite option. Meaning that only the changed items are downloaded over the network.)_.
  + Repeat loop until done.

_Note: There isn't any way to optimize the increments! Incrementing the transaction by more than 1 can mean that we miss a revert operation which could have been performed on a stream. It is important that we increment by *only* 1._

#### Deep-hist method ####

The third method is a little more complicated and requires an understanding of the `accurev hist` command and its caveats.

The `accurev hist` command when used to get the history for the stream only returns the transactions that occured in that stream.
However, a promotion into the parent stream could affect this stream and these transactions are *not* included in the ouput of the `accurev hist` command.

The _deep-hist method_ relies on creating a custom command for accurev that would return the set of all the transactions which could have possibly affected our stream.

This command is implemented in the `accurev.py` script. Here's a sample invocation:

```
import accurev
deepHistory = accurev.ext.deep_hist(depot="MyDepot", stream="MyStream", timeSpec="50-100")
print(deepHistory)
```

You can also use it directly by invocing the `accurev.py` script as follows:

```
python accurev.py deep-hist -p MyDepot -s MyStream -t 50-100
```

_Note: This command currently doesn't understand accurev time locks. This means that some transactions may be shown that do not have any affect on your stream because of a time lock._

Effectively this command does the heavy lifting for us so that the _diff method_ doesn't have to search through transactions one by one. Which finally brings us to how the _deep-hist method_ works:
 - Find the `mkstream` transaction and populate it.
 - Populate it in full and commit into git as an orphaned branch.
 - _Run the deep-hist function and get a list of transactions that affect this stream._
 - _Iterate over the transactions that deep-hist returned:_
  + Do an `accurev diff -a -i -v <stream> -V <stream>` between this transaction and the last transaction that we populated.
  + Delete only the files that `accurev diff` reported as changed from the git repository.
  + Populate the transaction and commit it into git. _(The populate here is done with the recursive option but without the overwrite option. Meaning that only the changed items are downloaded over the network.)_.
  + Repeat loop until done.

### Dear contributors ###

I am not a python developer which should be evident to anyone who's seen the code. A lot of it was written late at night and was meant to be just a brain dump, to be cleaned up at a later date, but it remained. Please don't be dissuaded from contributing and helping me improve it because it will get us all closer to ditching AccuRev! I will do my best to add some notes about my method and how the code works.

For now it works as I need it to and that's enough.

---
---

Copyright (c) 2015 Lazar Sumar

Permission is hereby granted, free of charge, to any person
obtaining a copy of this software and associated documentation
files (the "Software"), to deal in the Software without restriction,
including without limitation the rights to use, copy, modify, merge,
publish, distribute, sublicense, and/or sell copies of the Software,
and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

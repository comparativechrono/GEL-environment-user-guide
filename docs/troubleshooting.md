# Troubleshooting

[Back to the guide](../README.md)

**The page reloaded by itself the first time I opened it.**
That is expected: it installs a small helper that R and Python need, then reloads.

**RStudio or Spyder takes a long time to start.**
The first time, your browser downloads R or Python (some tens of megabytes). After that it keeps them, and they start faster. A table’s first query also takes a few seconds while the table loads.

**My files, projects or requests have gone.**
They are kept by your web browser, for this site, and for one username. Check that you are using the same browser, on the same computer, signed in with the same username, and not in a private window. If the browser’s data for the site were cleared, they are gone: this is why you export anything you need through the Airlock.

**I can’t paste something from my computer into the RE (or the other way).**
That is how the RE works: the clipboard stays inside. Type it in the RE, or bring the file in through the Airlock (see [Bringing files in](airlock.md#bringing-files-in)).

**A website I need is blocked.**
The RE has no internet, only a whitelist of reference sites (the **Whitelisted sites** bookmark lists them). Whitelisted sites open in a new tab of your own browser.

**`command not found` in the terminal.**
Load the program’s module first, for example `module load samtools/1.21`. `module avail` lists them.

**Rlabkey or labkey gives an error about signing in, or returns an HTML page.**
Set up `~/.netrc` with your username: see [The data](data.md#from-r). Check `baseUrl` (`https://labkey.prod.aws.gel.ac/labkey/`) and `folderPath`.

**`bsub` says “Please specify a project code” or “Project code … is not valid”.**
Add `-P re_gecip_training`, or the LSF code of your approved project.

**My Airlock export was rejected because my project was approved less than 30 days ago.**
That is the RE’s 30-day rule. On this course, export under the training project **RR4001**.

**My Airlock request was rejected for another reason.**
Open it under **My requests**: the reasons are listed there. Fix the files, then use **Start a new request with these files**. [The Airlock](airlock.md#the-automated-check) explains each message.

**Something is broken and reloading doesn’t help.**
Close the browser tab and open the simulator again. As a last resort, `?reset` at the end of the address deletes everything the simulator keeps in your browser and starts again: export anything you need first.

**Still stuck?**
Ask the course team: say what you did, what you expected, and what happened, with the exact error message.

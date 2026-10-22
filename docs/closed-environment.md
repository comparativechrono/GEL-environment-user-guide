# A closed environment

[Back to the guide](../README.md)

A trusted research environment (TRE) keeps the data in one secure place: you go to the data, rather than the data coming to you. The simulator copies the rules of the Genomics England RE, so that working this way becomes a habit before you meet real participants’ data.

## No internet

The RE has no general internet access. Its web browser reaches the RE’s own websites (LabKeeper, the Research Registry, the Airlock, the Variant Interpreter, the documentation) and a short **whitelist** of reference sites, such as PanelApp, ClinVar, gnomAD, Ensembl, OMIM, PubMed and HPO. Everything else shows the proxy’s “blocked” page. The **Whitelisted sites** bookmark lists them all.

In the simulator, whitelisted sites open in a tab of your own browser, because they cannot be shown inside another page.

There is no AI assistant in the RE, and no way to reach one: see [AI and reproducibility](ai-and-reproducibility.md).

## Copy and paste stay inside

Copy and paste work between the applications inside the RE: the terminal, the editors, RStudio, Spyder, IGV and the websites. But:

- what you copy inside the RE **cannot be pasted on your own computer**;
- what you copy on your own computer **cannot be pasted into the RE**: Ctrl+V pastes what you last copied inside;
- dragging text, pictures or files in or out does not work.

So write your code, notes and proposals **inside the RE**. If you draft something on your own computer, you will have to retype it, or bring the file in through the Airlock (see [Bringing files in](airlock.md#bringing-files-in)).

## Downloads stay inside

Downloads and exports, from LabKeeper, RStudio, Spyder or the terminal, are saved in `~/Downloads` **inside the RE**, not on your computer.

## One way out: the Airlock

Files leave the RE only through an Airlock export request, which is checked against the rules and reviewed. See [The Airlock](airlock.md). In the real RE you must not take screenshots of the RE, so get into the habit here: results leave as files, through the Airlock.

## Your folders

| Folder | What it is |
| --- | --- |
| `/home/<you>` (`~`) | Your home folder on the desktop. It is not visible on the HPC |
| `/re_gecip/training/<you>` | **Your project folder**: keep your work here. It is visible on the desktop and on the HPC. In the file manager it is the bookmark **my re_gecip folder** |
| `/re_scratch/re_gecip/training/<you>` | Temporary files |
| `/genomes`, `/gel_data_resources`, `/pgen_public_data_resources` | The data, read only. Your home folder has links to them |
| `/discovery_forum` | The industry members’ space: no access |

## The HPC

Larger jobs run on the high-performance cluster, which uses the LSF scheduler. In the simulator, jobs run in your browser, but with LSF’s commands, states and output files.

```bash
ssh jdoe@doublehelix.helix.prod.aws.gel.ac           # log in (use your username)
module load plink2/2.0.0-b.1-dev
bsub -q short -P re_gecip_training -o job.%J.out "plink2 --version"
bjobs                                                 # your jobs: PEND, RUN, DONE or EXIT
bpeek <job id>                                        # its output so far
exit                                                  # back to the desktop
```

| Queue | For |
| --- | --- |
| `inter` | Interactive work: `bsub -q inter -Is bash` |
| `short` | Jobs up to 4 hours |
| `medium` | Jobs up to 24 hours |
| `long` | Longer jobs |

The first time you log in, type `yes` to accept the HPC’s host key, then type any password (the user name must be your own). Your HPC home folder is not your desktop’s home, so keep your work in your project folder, which both can see. Graphical programs (RStudio, IGV …) run on the desktop only.

Every job needs a project code with `-P`. Use `re_gecip_training`, or the code your approved project was given in the Research Registry. Job scripts can carry the options on `#BSUB` lines, and `-w "done(<job id>)"` makes one job wait for another.

## Next

- [The Research Registry](research-registry.md)
- [The Airlock](airlock.md)

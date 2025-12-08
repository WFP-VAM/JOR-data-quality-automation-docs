# Installation

The application has several dependencies:

- R and Shiny for the main logic and visualizations.
- Python for the Data Bridges connection.
- DataBridgesKnots python package to interact with Data Bridges API.
- reticulate package to allow executing Python code.

So in order to run the application we need to have R and Python dependencies. This guide
covers the setup.

## Running it locally

### 1. System Dependencies

You will need the following software installed in your machine.

 - R: [https://www.r-project.org/](https://www.r-project.org/)
 - Python: [https://www.python.org/](https://www.python.org/downloads/)
 - Git: [https://git-scm.com/install/](https://git-scm.com/install/)
 - Rtools (optional for Windows): [https://cran.r-project.org/bin/windows/Rtools/rtools45/rtools.html](https://cran.r-project.org/bin/windows/Rtools/rtools45/rtools.html)

### 2. Checkout the repository.

Either using Github Application, R Studio, Positron IDE or the git command line, make a local copy of the
repository in your machine.

Link to the repository: [https://github.com/WFP-VAM/JOR-data-quality-automation](https://github.com/WFP-VAM/JOR-data-quality-automation)

Once you have a local copy of the project open it in your IDE.

### 3. Sync R Dependencies

This project uses [renv](https://rstudio.github.io/renv/) to manage the R dependencies so as soon as you
open the project in your IDE it will setup `renv` for you:

```
# Bootstrapping renv 1.1.5 ---------------------------------------------------
- Downloading renv ... OK
- Installing renv  ... OK

- Project 'C:/Users/pdelboca/Repos/JOR-data-quality-automation' loaded. [renv 1.1.5]
- One or more packages recorded in the lockfile are not installed.
- Use `renv::status()` for more details.
```

To install the R dependencies just execute the following command in the R Console:

```
renv:restore()
```

Installing the dependencies for the first time can take some time.

### 4. Install Python Dependencies

In order to install [DataBridgesKnots](https://github.com/WFP-VAM/DataBridgesKnots) we will need a [Python Virtual Environment](https://docs.python.org/3/library/venv.html).

So in the IDE, open the **Terminal** tab and run the following command to create a virtual environment

```bash
python -m venv .venv
```
**Note:** It is important that this command is executed in the root directory of the project. Running the application will
look for a folder called `.venv` and will fail if it does not find it.

After creating it, activate it:

```bash
source .venv/Scripts/activate # For windows Machines
```

**Note:** If running this command throws you an Execution Policy error try running the following command: `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` to unlock
the capacity to run scripts.

And finally install DataBridgesKnots requirement:

```bash
pip install "git+https://github.com/WFP-VAM/DataBridgesKnots.git"
```

### 5. Run the application

With the R environment restored and the Python dependencies updated you will now be able to run the application.


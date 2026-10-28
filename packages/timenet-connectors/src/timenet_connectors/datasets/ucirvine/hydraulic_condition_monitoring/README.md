# Writing a connector for Condition monitoring of hydraulic systems: Design decisions
I'll use this file to note my observations on the dataset and record the design decision taken along the way.

## Dataset
This dataset monitors the condition of an hydraulic test system. The engineers could set four components of the system to a chosen depletion level (cooler, valve, pump and accumulator) and it records 17 process signals during one cycle that last 60s.
For the same batch of settings, the engineers recorded multiple cycles in a row. These cycles that share common settings are called a run.

The content of the zip is
- one file per signal, with one row per cycle and one column per sample
- `profile.txt` which contains the four settings (cooler, valve, pump and accumulator) and a stable flag for each cycle
- `description.txt` contains the meaning of each setting's level, the names, the units and the sampling rates of the signal processes.
- `documentation.txt` similar as `description.txt`

In the connector, we pinned the `.zip` SHA256 so if the dataset content changes, the build fails and we know we need to adapt the connector and release a new version.

## Data investigation (~45')
Data is structured that way:
- The engineers choose the values for the 4 settings (cooler, valve, pump and accumulator). This defines a **run**
- For each run, the engineers launch consecutive **cycles** with the same four settings and they record/measure process values like pressures, volume flows and temperatures.
- Each cycle records values from 17 sensors


Looking at the data and how the signals correlate with each others and with the given hydraulic components condition, we can see interesting relationships:
- As expected, the Cooler Efficiency (CE), Cooling Power (CP) and the Cooler condition have a correlation of `> 0.95` with each other.
- TS1 to TS4 correalte at `1.00` with each other, it probably represents the temperature of the oil, at four different places. 
- Comparing per cycle by aggregating each signal to its mean so we can compare them together and also with the hydraulic component conditions, we see that the cooler condition has a strong positive correlation with FS2, PS3, PS5, PS6 and (obviously) CE and CP, a strong negative correlation with TS1 to TS4 and VS1, but doesn't really impact the efficiency factor (SE).

We stop here for the data investigation, we would find more useful information on the relationships between the signals if we had access to the paper.

dataset url: https://archive.ics.uci.edu/static/public/447/condition+monitoring+of+hydraulic+systems.zip

## Dataset modelling (~1h00)

Based on the investigation, we think it makes sense that one record is one cycle. A cycle is basically the smallest thing described in the give `profile.txt` file. One could argue that the labels are the same at the run level (except for the stable flag), but run as record means that we would have far less data and that we would lose the ability to use the model to understand one cycle. We would need an entire run before we can detect any drifts. Signals as record doesn't make sense for multiple reasons, but the main one is the signals are correlated with each other (multi-variate Time Series) so a model need to see all the signals of one cycle to make the best decision.

Once we set that one record is one set, we still have to decide if we want that each record holds 17 Sources with one Signal each or if one record should hold one Source with 17 Signals. As we didn't have access to the paper linked to the dataset, we didn't have any further information on the sources themselves and the `spec_type` attribute of the signal carries the information about what the signal measures (pressure, temperature, volume flow, ...), so we decided to go with latter option.

Each cycle gets four `ClassificationTask`, one per component setting (cooler, valve, pump, accumulator). As the levels are discrete settings the engineers have chosen, a regression task would predict something that wasn't used during the crafting of the dataset.
And four tasks so we can train a model for each one of them, so one per component, this way a customer can chose which components he wants to predict and also we can train one model per component and append the predicitons to have the same result as if we had only one task.

Then one debatable assumption regarding the target. If we want to use this dataset as-is to train an OpenTSLM model, then we can use as target the level's documented name instead of the level ("severe lag" instead of 80). I would argue that this probably won't have great results, but still could be useful to use as a baseline if we decide to generate a synthetic CoT dataset from this data and train an OpenTSLM model on it. 

## Dataset splitting 
There was no split shipped with the UCI dataset so we had to define one ourselves. The split is stored on the tasks and the four defined tasks in the previous section have the same data split.
- We decided to group the records by level combination (cooler, valve, pump, accumulator), instead of, for instance, run. The goal is that the training data gathers as many combinations as possible. Whereas if we had grouped by run, the training set would contain a lot of similar cycles, as cycles in the same run are almost copies. This way, we can also keep aside for the validation and the test sets totally unseen combination and thus we can see how the model manages to generalize.
- Accoring to the notes in `documentation.txt`, and from the correlation matrix we analyse during the data investigation, the `accumulator` variable is the hardest one to predict. So, to make sure that each partition of the split get the same mix of the `accumulator` variable, we use stratification on that label.
- After a cooling change, the system is not stable for a period of time, which impacts a few cycles. We decided not to include unstable records in the test set. As we think it's better to test the model on stable targets.

// 3a) Distinct presets for me (ID + Name with strong fallbacks)
ClearCollect(
    colMyPresets,
    ForAll(
        Distinct(colMySettings, Team_Preset_ID) As d,   // d.Result is the ID from User_Settings
        With(
            { pid: d.Result },
            {
                Preset_ID: pid,
                // Try to get a readable name from User_Settings first, then Team_Presets.Title (or Preset_Name if you have it),
                // finally fall back to the ID so we never return blank.
                Preset_Name:
                    Coalesce(
                        LookUp(colMySettings, Team_Preset_ID = pid, Team_Preset),
                        LookUp(Skill_Matrix_Team_Presets, Preset_ID = pid, Title),
                        LookUp(Skill_Matrix_Team_Presets, Preset_ID = pid, Preset_Name),
                        Text(pid)
                    )
            }
        )
    )
);

// 3b) Label text: "Preset A; Preset B; ..." (skip blanks just in case)
Set(
    varPresetListText,
    With(
        {
            names:
                SortByColumns(
                    Filter(colMyPresets, !IsBlank(Preset_Name)),
                    "Preset_Name",
                    Ascending
                )
        },
        If(
            CountRows(names) = 0,
            "None assigned",
            Concat(names, Preset_Name, "; ")
        )
    )
);

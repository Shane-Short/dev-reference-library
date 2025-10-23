// 3a) Distinct presets for me (ID + Name with strong fallbacks)
ClearCollect(
    colMyPresets,
    AddColumns(
        Distinct(colMySettings, Team_Preset_ID),
        Preset_ID, ThisRecord.Result,
        Preset_Name,
            // 1) Try User_Settings.Team_Preset (your current label)
            Coalesce(
                LookUp(colMySettings, Team_Preset_ID = ThisRecord.Result, Team_Preset),
                // 2) Try Team_Presets.Title (you said Title holds the preset name there)
                LookUp(Skill_Matrix_Team_Presets, Preset_ID = ThisRecord.Result, Title),
                // 3) Try Team_Presets.Preset_Name if it exists
                LookUp(Skill_Matrix_Team_Presets, Preset_ID = ThisRecord.Result, Preset_Name),
                // 4) Fallback to the ID (so we never get empty)
                Text(ThisRecord.Result)
            )
    )
);

// 3b) Label text: "Preset A; Preset B; ..." (skip blanks, just in case)
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

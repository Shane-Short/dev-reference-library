// ---------- Who am I ----------
Set(varMe, Lower(Trim(User().Email)));

// ---------- My User_Settings rows ----------
ClearCollect(
    colMySettings,
    Filter(Skill_Matrix_User_Settings, Lower(Trim(Employee_Email)) = varMe)
);

// Guard: if no settings, keep collections empty and exit early
If( CountRows(colMySettings) = 0,
    Clear(colMyPresets); Clear(colMyModules); Clear(colPresetItems); Clear(colUserEntries); Clear(colUserEntriesNorm); Clear(colUserSelections);
    Set(varPresetListText, "None assigned");
    // early return pattern (no-op)
    false,
    
    // ---------- All my presets (distinct) ----------
    ClearCollect(
        colMyPresets,
        AddColumns(
            // Distinct by Preset_ID, but keep the readable name
            ForAll(
                Distinct(colMySettings, Team_Preset_ID),
                {
                    Preset_ID: ThisRecord.Value,
                    Preset_Name: LookUp(colMySettings, Team_Preset_ID = ThisRecord.Value).Team_Preset
                }
            ),
            // tidy display text if you want
            "Display", Preset_Name
        )
    );

    // Text shown on the page: "Team Preset(s): A; B; C"
    Set(
        varPresetListText,
        If(
            CountRows(colMyPresets)=0,
            "None assigned",
            Concat( SortByColumns(colMyPresets, "Preset_Name", Ascending), Preset_Name, "; " )
        )
    );

    // ---------- All modules across ALL my presets ----------
    // Team_Presets is normalized: one row per preset + module
    ClearCollect(
        colMyModules,
        AddColumns(
            Distinct(
                Filter(
                    Skill_Matrix_Team_Presets,
                    CountIf(colMyPresets, Preset_ID = Team_Preset_ID) > 0
                ),
                Modules
            ),
            "Title", ThisRecord.Value
        )
    );

    // ---------- All reference rows for those modules ----------
    // Use normalized compare to be resilient to casing/spaces
    ClearCollect(
        colPresetItems,
        With(
            {
                mods: AddColumns(colMyModules, nTitle, Lower(Trim(Title)))
            },
            Filter(
                Skill_Matrix_Reference As r,
                CountIf(mods, nTitle = Lower(Trim(r.Module))) > 0
            )
        )
    );

    // ---------- My existing entries ----------
    ClearCollect(
        colUserEntries,
        Filter(Skill_Matrix_Entries, Lower(Trim(Employee_Email)) = varMe)
    );

    // Normalized copy for robust matching
    ClearCollect(
        colUserEntriesNorm,
        AddColumns(
            colUserEntries,
            nModule,   Lower(Trim(Module)),
            nCategory, Lower(Trim(Category)),
            nItem,     Lower(Trim(Item))
        )
    );

    // ---------- Build the working set the UI uses ----------
    // (one row per Module+Category+Item with the user's current Skill_Level)
    ClearCollect(
        colUserSelections,
        AddColumns(
            AddColumns(
                colPresetItems,
                nModule,   Lower(Trim(Module)),
                nCategory, Lower(Trim(Category)),
                nItem,     Lower(Trim(Item))
            ),
            Skill_Level,
                Coalesce(
                    LookUp(
                        colUserEntriesNorm,
                        nModule = Self.nModule && nCategory = Self.nCategory && nItem = Self.nItem,
                        Skill_Level
                    ),
                    1   // default level when no entry exists
                ),
            HasChanged, false
        )
    )
);

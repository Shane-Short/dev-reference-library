// 1) Who am I (robust)
Set(
    varMe,
    Lower(
        Trim(
            Coalesce(
                Office365Users.MyProfileV2().mail,
                User().Email
            )
        )
    )
);

// 2) My settings rows
ClearCollect(
    colMySettings,
    Filter(Skill_Matrix_User_Settings, Lower(Trim(Employee_Email)) = varMe)
);

// 3) If I have no settings, clear and exit; else build state
If(
    CountRows(colMySettings) = 0,
    Clear(colMyPresets);
    Clear(colMyModules);
    Clear(colPresetItems);
    Clear(colUserEntries);
    Clear(colUserEntriesNorm);
    Clear(colUserSelections);
    Set(varPresetListText, "None assigned"),
    
    // --- Distinct presets for me (ID + Name from Team_Preset) ---
    ClearCollect(
        colMyPresets,
        AddColumns(
            Distinct(colMySettings, Team_Preset_ID),
            "Preset_ID", ThisRecord.Result,
            "Preset_Name",
                LookUp(
                    colMySettings,
                    Team_Preset_ID = ThisRecord.Result,
                    Team_Preset
                )
        )
    );

    // --- Label text: "Preset A; Preset B; ..." ---
    Set(
        varPresetListText,
        If(
            CountRows(colMyPresets) = 0,
            "None assigned",
            Concat(
                SortByColumns(colMyPresets, "Preset_Name", Ascending),
                Preset_Name,
                "; "
            )
        )
    );

    // --- All modules across ALL my presets (Team_Presets: 1 row per Preset+Module) ---
    ClearCollect(
        colMyModules,
        AddColumns(
            Distinct(
                Filter(
                    Skill_Matrix_Team_Presets As tp,
                    CountIf(colMyPresets, Preset_ID = tp.Preset_ID) > 0
                ),
                Modules
            ),
            "Title", ThisRecord.Result
        )
    );

    // --- All reference rows (Module/Category/Item) for those modules ---
    ClearCollect(
        colPresetItems,
        With(
            { mods: AddColumns(colMyModules, "nTitle", Lower(Trim(Title))) },
            Filter(
                Skill_Matrix_Reference As r,
                CountIf(mods, nTitle = Lower(Trim(r.Module))) > 0
            )
        )
    );

    // --- My existing entries ---
    ClearCollect(
        colUserEntries,
        Filter(Skill_Matrix_Entries, Lower(Trim(Employee_Email)) = varMe)
    );

    // --- Normalized copy of my entries (for reliable matching) ---
    ClearCollect(
        colUserEntriesNorm,
        AddColumns(
            colUserEntries,
            "nModule",   Lower(Trim(Module)),
            "nCategory", Lower(Trim(Category)),
            "nItem",     Lower(Trim(Item))
        )
    );

    // --- Working set for the UI: every preset item + current Skill_Level (default 1) ---
    ClearCollect(
        colUserSelections,
        AddColumns(
            // normalize keys on the reference rows
            AddColumns(
                colPresetItems,
                "nModule",   Lower(Trim(Module)),
                "nCategory", Lower(Trim(Category)),
                "nItem",     Lower(Trim(Item))
            ),
            // inject Skill_Level by looking up normalized entry; default to 1
            "Skill_Level",
                Coalesce(
                    LookUp(
                        colUserEntriesNorm As ue,
                        ue.nModule = ThisRecord.nModule &&
                        ue.nCategory = ThisRecord.nCategory &&
                        ue.nItem = ThisRecord.nItem,
                        ue.Skill_Level
                    ),
                    1
                ),
            "HasChanged", false
        )
    )
);

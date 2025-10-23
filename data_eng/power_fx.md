// ---------- 1) Who am I (robust) ----------
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

// Optional while stabilizing data:
Refresh(Skill_Matrix_User_Settings);

// ---------- 2) My settings rows ----------
ClearCollect(
    colMySettings,
    Filter(Skill_Matrix_User_Settings, Lower(Trim(Employee_Email)) = varMe)
);

// ---------- 3) If I have no settings, clear and exit ----------
If(
    CountRows(colMySettings) = 0,
    // Clear downstream collections and label
    Clear(colMyPresets);
    Clear(colMyModules);
    Clear(colPresetItems);
    Clear(colUserEntries);
    Clear(colUserEntriesNorm);
    Clear(colUserSelections);
    Set(varPresetListText, "None assigned"),
    
    // ---------- ELSE: build all state ----------
    // 3a) Distinct presets for me (ID + Name)
    ClearCollect(
        colMyPresets,
        AddColumns(
            Distinct(colMySettings, Team_Preset_ID),
            "Preset_ID", ThisRecord.Value,
            "Preset_Name",
                LookUp(
                    colMySettings,
                    Team_Preset_ID = ThisRecord.Value,
                    Coalesce(Team_Preset, Preset_Name, Title)
                )
        )
    );

    // 3b) Label text: "Preset A; Preset B; ..."
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

    // 3c) All modules across ALL my presets (Team_Presets is 1 row per Preset+Module)
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
            "Title", ThisRecord.Result
        )
    );

    // 3d) All reference rows (Module/Category/Item) for those modules
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

    // 3e) My existing entries
    ClearCollect(
        colUserEntries,
        Filter(Skill_Matrix_Entries, Lower(Trim(Employee_Email)) = varMe)
    );

    // 3f) Normalized copy of my entries (for reliable matching)
    ClearCollect(
        colUserEntriesNorm,
        AddColumns(
            colUserEntries,
            "nModule",   Lower(Trim(Module)),
            "nCategory", Lower(Trim(Category)),
            "nItem",     Lower(Trim(Item))
        )
    );

    // 3g) Working set for the UI: every preset item + current Skill_Level (default 1)
    ClearCollect(
        colUserSelections,
        AddColumns(
            // first normalize keys on the reference rows
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

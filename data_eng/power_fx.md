// ---- My existing entries (raw) ----
ClearCollect(
    colUserEntries,
    Filter(Skill_Matrix_Entries, Lower(Trim(Employee_Email)) = varMe)
);

// ---- Normalized copy for robust matching ----
ClearCollect(
    colUserEntriesNorm,
    AddColumns(
        colUserEntries,
        "nModule",   Lower(Trim(Module)),
        "nCategory", Lower(Trim(Category)),
        "nItem",     Lower(Trim(Item))
    )
);

// ---- All reference rows for my modules (you already built colMyModules above) ----
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

// ---- Working set used by the UI (adds user's current Skill_Level) ----
ClearCollect(
    colUserSelections,
    AddColumns(
        // first add normalized keys to the reference rows
        AddColumns(
            colPresetItems,
            "nModule",   Lower(Trim(Module)),
            "nCategory", Lower(Trim(Category)),
            "nItem",     Lower(Trim(Item))
        ),
        // now derive Skill_Level by looking up in the normalized entries
        "Skill_Level",
            Coalesce(
                LookUp(
                    colUserEntriesNorm As ue,
                    ue.nModule = ThisRecord.nModule
                        && ue.nCategory = ThisRecord.nCategory
                        && ue.nItem = ThisRecord.nItem,
                    ue.Skill_Level
                ),
                1   // default when no entry yet
            ),
        "HasChanged", false
    )
);

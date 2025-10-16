// 6) OPTIONAL: Keep existing Reference rows' Skill_Type synced with CategoryItems (all Text)

// Take a local snapshot of the Reference rows for this module (avoid patching the same DS you're iterating)
ClearCollect(
    colRefToSync,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = moduleId),
        "ID",          // SharePoint built-in ID
        "CatItem_ID",
        "Skill_Type"
    )
);

Clear(colDidUpdate);

ForAll(
    colRefToSync As r,
    With(
        {
            ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = r.CatItem_ID)
        },
        If(
            !IsBlank(ci) &&
            !IsBlank(ci.Skill_Type) &&
            ci.Skill_Type <> r.Skill_Type,
            // Patch by looking up the live record via its ID
            Patch(
                Skill_Matrix_Reference,
                LookUp(Skill_Matrix_Reference, ID = r.ID),
                { Skill_Type: ci.Skill_Type }
            );
            Collect(colDidUpdate, { id: r.CatItem_ID })
        )
    )
);

// (optional) Clear the snapshot if you like
// Clear(colRefToSync);

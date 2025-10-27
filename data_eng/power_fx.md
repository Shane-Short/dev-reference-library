// 2.2 Remove any links that are no longer checked
ForAll(
    Filter(
        Skill_Matrix_Reference,
        Mod_ID = varSelectedModuleId &&
        CountIf(colModuleCatItems_Working, CatItem_ID = CatItem_ID) = 0
    ),
    Remove(Skill_Matrix_Reference, ThisRecord)
);

// 2.3 (Optional) keep Skill_Type synced with CategoryItems for all links in this module
ForAll(
    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
    With(
        { ci2: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ThisRecord.CatItem_ID) },
        If(
            !IsBlank(ci2) && ci2.Skill_Type <> ThisRecord.Skill_Type,
            Patch(Skill_Matrix_Reference, ThisRecord, { Skill_Type: ci2.Skill_Type })
        )
    )
);

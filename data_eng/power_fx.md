// 2.3 (Optional) keep Skill_Type synced with CategoryItems for all links in this module
// Project to a known shape so Skill_Type is recognized
ClearCollect(
    colRefsForSync,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
        "Reference_ID", "CatItem_ID", "Skill_Type"
    )
);

ForAll(
    colRefsForSync As ref,
    With(
        { ci2: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ref.CatItem_ID) },
        If(
            !IsBlank(ci2) && ci2.Skill_Type <> ref.Skill_Type,
            Patch(
                Skill_Matrix_Reference,
                LookUp(Skill_Matrix_Reference, Reference_ID = ref.Reference_ID),
                { Skill_Type: ci2.Skill_Type }
            )
        )
    )
);

// (optional) cleanup
Clear(colRefsForSync)

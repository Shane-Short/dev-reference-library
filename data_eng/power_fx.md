// 6) OPTIONAL: keep existing Reference rows' Skill_Type synced with CategoryItems (all Text)
Clear(colDidUpdate);

ForAll(
    Filter(Skill_Matrix_Reference, Mod_ID = moduleId) As ref,   // alias belongs to ForAll
    With(
        {
            ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ref.CatItem_ID)
        },
        If(
            !IsBlank(ci) &&
            !IsBlank(ci.Skill_Type) &&
            ci.Skill_Type <> ref.Skill_Type,
            Patch(Skill_Matrix_Reference, ref, { Skill_Type: ci.Skill_Type });
            Collect(colDidUpdate, { id: ref.CatItem_ID })
        )
    )
);

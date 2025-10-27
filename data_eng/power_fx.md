// 2.3 (Optional) keep Skill_Type synced with CategoryItems for all links in this module
ForAll(
    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
    With(
        { ci2: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ThisRecord.CatItem_ID) },
        If(
            !IsBlank(ci2) && ci2.Skill_Type <> ThisRecord.SkillLevel,
            Patch(Skill_Matrix_Reference, ThisRecord, { SkillLevel: ci2.Skill_Type })
        )
    )
);

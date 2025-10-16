
// 6) OPTIONAL: Keep existing Reference rows' Skill_Type synced with CategoryItems
Clear(colDidUpdate);

ForAll(
    Filter(Skill_Matrix_Reference As ref, ref.Mod_ID = moduleId),
    With(
        {
            ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ref.CatItem_ID),

            // Normalize types: extract text regardless of Choice/Text column types
            ciType: If(
                IsBlank(ci),
                Blank(),
                If(IsRecord(ci.Skill_Type), ci.Skill_Type.Value, ci.Skill_Type)
            ),
            refType: If(IsRecord(ref.Skill_Type), ref.Skill_Type.Value, ref.Skill_Type),

            // Will Reference.Skill_Type accept a Choice or Text? Detect and format
            newRefSkillType: If(
                IsRecord(ref.Skill_Type),
                { Value: ciType },      // Reference is a Choice column
                ciType                  // Reference is a Text column
            )
        },
        If(
            !IsBlank(ci) && ciType <> refType,
            Patch(Skill_Matrix_Reference, ref, { Skill_Type: newRefSkillType });
            Collect(colDidUpdate, { id: ref.CatItem_ID })
        )
    )
);

// Normalize Skill_Type when adding
Skill_Type: If(
    IsRecord(First(Skill_Matrix_Reference).Skill_Type),
    { Value: If(IsRecord(ci.Skill_Type), ci.Skill_Type.Value, ci.Skill_Type) },   // Reference is Choice
    If(IsRecord(ci.Skill_Type), ci.Skill_Type.Value, ci.Skill_Type)               // Reference is Text
),

// =====================
// 2) WRITE CHANGES to Reference (adds, removes, sync types)
// =====================

// 2.1 Add any newly-checked CatItems for this module
ForAll(
    Filter(
        colModuleCatItems_Working As w,
        IsBlank(
            LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && CatItem_ID = w.CatItem_ID
            )
        )
    ),
    With(
        {
            ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = w.CatItem_ID),
            modName: varSelectedModuleName
        },
        Patch(
            Skill_Matrix_Reference,
            Defaults(Skill_Matrix_Reference),
            {
                Reference_ID: GUID(),
                Mod_ID: varSelectedModuleId,
                CatItem_ID: w.CatItem_ID,

                // Denormalized text (keeps reports/UI happy)
                Module: modName,
                Category: ci.Category,
                Item: ci.Item,
                Skill_Type: ci.Skill_Type,

                Created_By: User().Email,
                Created_At: Now()
            }
        )
    )
);

// 2.2 Remove any links that are no longer checked
ForAll(
    Filter(
        Skill_Matrix_Reference As r,
        r.Mod_ID = varSelectedModuleId &&
        CountIf(colModuleCatItems_Working, CatItem_ID = r.CatItem_ID) = 0
    ),
    Remove(Skill_Matrix_Reference, r)
);

// 2.3 (Optional) keep Skill_Type synced with CategoryItems for all links in this module
ForAll(
    Filter(Skill_Matrix_Reference As r, r.Mod_ID = varSelectedModuleId),
    With(
        { ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = r.CatItem_ID) },
        If(
            !IsBlank(ci) && ci.Skill_Type <> r.Skill_Type,
            Patch(Skill_Matrix_Reference, r, { Skill_Type: ci.Skill_Type })
        )
    )
);

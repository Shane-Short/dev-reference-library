// =====================
// 2) WRITE CHANGES to Reference (adds, removes, sync types)
// =====================

// 2.1 Add any newly-checked CatItems for this module
ForAll(
    // Work only with rows that *have* a CatItem_ID and aren't already linked
    Filter(
        colModuleCatItems_Working As row,
        !IsBlank(row.CatItem_ID) &&
        IsBlank(
            LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && CatItem_ID = row.CatItem_ID
            )
        )
    ),
    With(
        {
            id: row.CatItem_ID, // capture once to avoid alias scope weirdness
            ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = row.CatItem_ID)
        },
        // only Patch if we could resolve the Category/Item (defensive)
        If(
            !IsBlank(id) && !IsBlank(ci),
            Patch(
                Skill_Matrix_Reference,
                Defaults(Skill_Matrix_Reference),
                {
                    Reference_ID: GUID(),
                    Mod_ID: varSelectedModuleId,
                    CatItem_ID: id,

                    // denormalized text (keeps reports/UI happy)
                    Module:    varSelectedModuleName,
                    Category:  ci.Category,
                    Item:      ci.Item,
                    Skill_Type: ci.Skill_Type,

                    Created_By: User().Email,
                    Created_At: Now()
                }
            )
        )
    )
);

// 2.2 Remove any links that are no longer checked
ForAll(
    Filter(
        Skill_Matrix_Reference As ref,
        ref.Mod_ID = varSelectedModuleId &&
        CountIf(colModuleCatItems_Working, CatItem_ID = ref.CatItem_ID) = 0
    ),
    Remove(Skill_Matrix_Reference, ref)
);

// 2.3 (Optional) keep Skill_Type synced with CategoryItems for all links in this module
ForAll(
    Filter(Skill_Matrix_Reference As ref, ref.Mod_ID = varSelectedModuleId),
    With(
        { ci2: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ref.CatItem_ID) },
        If(
            !IsBlank(ci2) && ci2.Skill_Type <> ref.Skill_Type,
            Patch(Skill_Matrix_Reference, ref, { Skill_Type: ci2.Skill_Type })
        )
    )
);

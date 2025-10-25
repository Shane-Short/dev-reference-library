Added_CatItem_IDs:
    Concat(
        ForAll(
            colDidAdd As a,
            Coalesce(
                // 1) if your cDidAdd has Reference_ID
                LookUp(Skill_Matrix_Reference, Reference_ID = a.Reference_ID, CatItem_ID),
                // 2) else try by module+category+item if those exist on 'a'
                LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Category = a.Category && Item = a.Item, CatItem_ID),
                // 3) else try CategoryItems by Category+Item
                LookUp(Skill_Matrix_CategoryItems, Category = a.Category && Item = a.Item, CatItem_ID)
            )
        ),
        Text(Value),
        ";"
    ),
Removed_CatItem_IDs:
    Concat(
        ForAll(
            colDidRemove As r,
            Coalesce(
                LookUp(Skill_Matrix_Reference, Reference_ID = r.Reference_ID, CatItem_ID),
                LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Category = r.Category && Item = r.Item, CatItem_ID),
                LookUp(Skill_Matrix_CategoryItems, Category = r.Category && Item = r.Item, CatItem_ID)
            )
        ),
        Text(Value),
        ";"
    ),


// when you detect an ADD
Collect(colDidAdd, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

// when you detect a REMOVE
Collect(colDidRemove, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

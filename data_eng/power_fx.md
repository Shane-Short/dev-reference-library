// compute added/removed CatItem_IDs by comparing Reference (before) vs Working (after)
With(
    {
        beforeTbl: Filter(Skill_Matrix_Reference As r, r.Mod_ID = varSelectedModuleId),
        afterTbl:  colModuleCatItems_Working
    },
    // serialize added IDs
    Added_CatItem_IDs:
        Concat(
            Filter(
                afterTbl As w,
                CountIf(beforeTbl, CatItem_ID = w.CatItem_ID) = 0
            ),
            CatItem_ID,
            ";"
        ),
    // serialize removed IDs
    Removed_CatItem_IDs:
        Concat(
            Filter(
                beforeTbl As r,
                CountIf(afterTbl, CatItem_ID = r.CatItem_ID) = 0
            ),
            CatItem_ID,
            ";"
        ),
    // counts that match those lists
    Added_Count:
        CountRows(
            Filter(
                afterTbl As w,
                CountIf(beforeTbl, CatItem_ID = w.CatItem_ID) = 0
            )
        ),
    Removed_Count:
        CountRows(
            Filter(
                beforeTbl As r,
                CountIf(afterTbl, CatItem_ID = r.CatItem_ID) = 0
            )
        )
)


// when you detect an ADD
Collect(colDidAdd, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

// when you detect a REMOVE
Collect(colDidRemove, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

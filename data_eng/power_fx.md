With(
    {
        addIds: ForAll(
                    colDidAdd As a,
                    LookUp(Skill_Matrix_Reference, Reference_ID = a.Reference_ID, CatItem_ID)
                ),
        remIds: ForAll(
                    colDidRemove As r,
                    LookUp(Skill_Matrix_Reference, Reference_ID = r.Reference_ID, CatItem_ID)
                )
    },
    Added_CatItem_IDs:   Concat(addIds, Text(Value), ";"),
    Removed_CatItem_IDs: Concat(remIds, Text(Value), ";")
)




// when you detect an ADD
Collect(colDidAdd, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

// when you detect a REMOVE
Collect(colDidRemove, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

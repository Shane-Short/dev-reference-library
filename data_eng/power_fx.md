// Build CatItem universe (limit to selected category or include all)
ClearCollect(
    colCIUniverse,
    ShowColumns(
        If(
            !IsBlank(drpCategory.Selected.Value),
            Filter(Skill_Matrix_CategoryItems, Category = drpCategory.Selected.Value),
            Skill_Matrix_CategoryItems
        ),
        "CatItem_ID"
    )
);

// Unique CatItem_IDs
ClearCollect(
    colCIIds,
    ShowColumns(
        GroupBy(colCIUniverse, "CatItem_ID", "grp"),
        "CatItem_ID"
    )
);

// BEFORE snapshot of Skill_Type per CatItem_ID
ClearCollect(
    colCIBefore,
    ForAll(
        colCIIds As t,
        {
            CatItem_ID: t.CatItem_ID,
            Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = t.CatItem_ID, Skill_Type)
        }
    )
);

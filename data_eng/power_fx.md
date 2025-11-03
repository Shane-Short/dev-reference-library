// Rebuild the working set for the chosen Category,
// and pre-mark each row as selected if it's active in Reference
ClearCollect(
    colModuleCatItems_Working,
    With(
        {
            // 1) Normalize Category to text safely for BOTH Choice or Text columns
            src: AddColumns(
                    Skill_Matrix_CategoryItems,
                    CategoryText, Text(Category)   // <-- key change: no ".Value"
                 )
        },
        AddColumns(
            // 2) Keep only the chosen category
            Filter(
                src,
                CategoryText = Text(cmbSelectCategory.Selected.Value)
            ),
            // 3) Precheck flag (used by checkbox Default)
            IsSelectedForModule,
            CountIf(
                colRefActiveIds,
                CatItem_ID = ThisRecord.CatItem_ID
            ) > 0
        )
    )
);

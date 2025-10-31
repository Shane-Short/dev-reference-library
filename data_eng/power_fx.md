// 6.3) Mark refs that should no longer be in the module as inactive
ClearCollect(
    colRefsToRemove,
    ForAll(
        colRefBeforeSave As b,
        If(
            // if b.CatItem_ID is NOT in the target list, it should be disabled
            CountIf(colTargetCatItems, CatItem_ID = b.CatItem_ID) = 0,

            // pull the full ref row from the list so Patch can target it
            LookUp(
                Skill_Matrix_Reference,
                Reference_ID = b.Reference_ID
            )
        )
    )
);

// Now soft-delete (flip IsActive) instead of hard delete
ForAll(
    colRefsToRemove As deadRef,
    Patch(
        Skill_Matrix_Reference,
        LookUp(
            Skill_Matrix_Reference,
            Reference_ID = deadRef.Reference_ID
        ),
        {
            IsActive: false,
            Updated_By: reqUserEmail,
            Updated_At: Now()
        }
    )
);

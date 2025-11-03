// 6a.1) Build rows we must insert (for ids in colAdds that lacked an existing ref)
ClearCollect(
    colAdd_NewRows,
    ForAll(
        colAdds As a,
        With(
            { existingRef: LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id) },
            If(
                IsBlank(existingRef),
                {
                    Mod_ID:     varSelectedModuleId,
                    CatItem_ID: IfError(Value(a.id), a.id),   // works whether CatItem_ID is Number or Text
                    IsActive:   true,
                    CreatedBy:  reqUserEmail,
                    CreatedAt:  Now()
                }
            )
        )
    )
);
// Remove blanks (when existingRef was found)
ClearCollect(colAdd_NewRows, Filter(colAdd_NewRows, !IsBlank(Mod_ID)));










Set(varAddCount,    CountRows(Coalesce(colToAdd, Table())));
Set(varRemoveCount, CountRows(Coalesce(colToRemove, Table())));
Set(varUpdateCount, CountRows(Coalesce(colTypeChanges, Table())));









Set(varUpdCount,     CountRows(colCI_Changed));









// cache refs for this module once
ClearCollect(
    colRefAllByModule,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
        "CatItem_ID", "Reference_ID", "IsActive"
    )
);








Patch(
    colToggleLog,
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID),
    { CatItem_ID: ThisItem.CatItem_ID, Desired: true }
);







Patch(
    colToggleLog,
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID),
    { CatItem_ID: ThisItem.CatItem_ID, Desired: false }
);








// 4) Desired selection from global toggle log (all categories)
ClearCollect(
    colDesiredSel,
    AddColumns(
        ShowColumns(Filter(colToggleLog, Desired = true), "CatItem_ID"),
        id, Text(CatItem_ID)
    )
);

// 5) Global diffs vs current active refs (no cat filter)
ClearCollect(
    colAdds,
    Filter(colDesiredSel As d, CountIf(colRefActiveIds, id = d.id) = 0)
);

ClearCollect(
    colRemoves,
    Filter(colRefActiveIds As a, CountIf(colDesiredSel, id = a.id) = 0)
);











Clear(colToggleLog);







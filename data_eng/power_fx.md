// 1) Snapshot: ACTIVE refs for this module as TEXT ids
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            "CatItem_ID"
        ),
        id, Text(CatItem_ID)         // normalize to text id
    )
);

// 2) Category’s CatItems (normalized id + curSelected using the same id)
ClearCollect(
    colCatBase,
    AddColumns(
        Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
        id, Text(CatItem_ID),
        curSelected, CountIf(colRefActiveIds, id = Text(ThisRecord.CatItem_ID)) > 0
    )
);

// 3) Desired = overlay local toggle log on top of curSelected (compare by the SAME id)
ClearCollect(
    colDesired,
    AddColumns(
        colCatBase,
        Desired,
            With(
                { tgl: LookUp(colToggleLog, CatItem_ID = id) },   // <— compare by id
                If(IsBlank(tgl), curSelected, tgl.Desired)
            )
    )
);

// 4) Diffs (adds/removes)
ClearCollect(colAdds,    Filter(colDesired, Desired = true  && curSelected = false));
ClearCollect(colRemoves, Filter(colDesired, Desired = false && curSelected = true));









// 5) APPLY ADDS
ForAll(
    colAdds As a,
    With(
        {
            existingRef: LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id
            )
        },
        If(
            !IsBlank(existingRef),
            Patch(Skill_Matrix_Reference, existingRef, { IsActive: true, UpdatedBy: reqUserEmail, UpdatedAt: Now() }),
            With(
                { newRefId: GUID() },
                Patch(
                    Skill_Matrix_Reference,
                    Defaults(Skill_Matrix_Reference),
                    {
                        Reference_ID: newRefId,
                        Mod_ID: varSelectedModuleId,
                        Module: varSelectedModuleName,
                        CatItem_ID: Value(a.id),   // if CatItem_ID is numeric; else just a.id
                        Category: a.Category,
                        Item: a.Item,
                        SkillLevel: a.Skill_Type,
                        IsActive: true,
                        Created_By: reqUserEmail,
                        Created_At: Now()
                    }
                )
            )
        )
    )
);

// 6) APPLY REMOVES
ForAll(
    colRemoves As r,
    With(
        {
            activeRef: LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && Text(CatItem_ID) = r.id && IsActive = true
            )
        },
        If(
            !IsBlank(activeRef),
            Patch(Skill_Matrix_Reference, activeRef, { IsActive: false, UpdatedBy: reqUserEmail, UpdatedAt: Now() })
        )
    )
);





// DEBUG: show what will change
ClearCollect(colDiffs, Filter(colDesired, Desired <> curSelected));
Set(varToggles, CountRows(colToggleLog));
Set(varDiffCount, CountRows(colDiffs));
// Optional: Notify("toggles=" & varToggles & " diffs=" & varDiffCount, NotificationType.Information);




If(
  IsBlank(LookUp(colToggleLog, CatItem_ID = Text(ThisItem.CatItem_ID))),
  Collect(colToggleLog, { CatItem_ID: Text(ThisItem.CatItem_ID), Desired: true }),
  UpdateIf(colToggleLog, CatItem_ID = Text(ThisItem.CatItem_ID), { Desired: true })
)





If(
  IsBlank(LookUp(colToggleLog, CatItem_ID = Text(ThisItem.CatItem_ID))),
  Collect(colToggleLog, { CatItem_ID: Text(ThisItem.CatItem_ID), Desired: false }),
  UpdateIf(colToggleLog, CatItem_ID = Text(ThisItem.CatItem_ID), { Desired: false })
)






// hard reset of scratch vars/collections with schema-preserving empties
Set(varSelectedModuleId, Blank());
Set(varSelectedModuleName, Blank());
Set(varSelectedCategoryText, Blank());

// empty toggle log with explicit columns
ClearCollect(
    colToggleLog,
    FirstN(
        Table({ id: "", Desired: false }),
        0
    )
);

// empty active refs (id = text CatItem_ID)
ClearCollect(
    colRefActiveIds,
    FirstN(
        Table({ id: "" }),
        0
    )
);

// empty working list but with the same columns the gallery expects
ClearCollect(
    colModuleCatItems_Working,
    FirstN(
        Table({
            CatItem_ID: "",
            Category: "",
            Item: "",
            Skill_Type: "",
            id: "",
            IsSelectedForModule: false
        }),
        0
    )
);




// cache selected module (use your control names)
Set(varSelectedModuleId, cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// refresh active refs for this module
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            "CatItem_ID"
        ),
        id, Text(CatItem_ID)
    )
);

// clear any pending toggles when module changes
Clear(colToggleLog);

// rebuild the working set if a category is already chosen
If(
    !IsBlank(varSelectedCategoryText),
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Text(Category) = varSelectedCategoryText
                ),
                id, Text(CatItem_ID)
            ),
            IsSelectedForModule,
            CountIf(colRefActiveIds, id = ThisRecord.id) > 0
        )
    )
)










// cache text safely for Choice/Text categories
Set(
    varSelectedCategoryText,
    If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value))
);

// refresh active refs again (cheap + safe)
If(
    !IsBlank(varSelectedModuleId),
    ClearCollect(
        colRefActiveIds,
        AddColumns(
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                "CatItem_ID"
            ),
            id, Text(CatItem_ID)
        )
    )
);

// clear toggles when category changes
Clear(colToggleLog);

// build working list for this category with correct check-state
If(
    IsBlank(varSelectedModuleId) || IsBlank(varSelectedCategoryText),
    Clear(colModuleCatItems_Working),
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Text(Category) = varSelectedCategoryText
                ),
                id, Text(CatItem_ID)
            ),
            IsSelectedForModule,
            CountIf(colRefActiveIds, id = ThisRecord.id) > 0
        )
    )
)













// always returns a table with id + IsSelectedForModule so schema is stable
With(
    { catText: varSelectedCategoryText },
    With(
        {
            baseRows:
                If(
                    IsBlank(varSelectedModuleId) || IsBlank(catText),
                    FirstN(Filter(Skill_Matrix_CategoryItems, false), 0),
                    Filter(Skill_Matrix_CategoryItems, Text(Category) = catText)
                )
        },
        AddColumns(
            AddColumns(baseRows, id, Text(CatItem_ID)),
            IsSelectedForModule,
            CountIf(colRefActiveIds, id = ThisRecord.id) > 0
        )
    )
)










// ======================= Save Module Config (stable) =======================
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

If(
    IsBlank(varSelectedModuleId) || IsBlank(varSelectedCategoryText),
    Notify("Pick a module and a category first.", NotificationType.Warning),

    With(
        { catText: varSelectedCategoryText },

        // 1) current active refs (id=text CatItem_ID)
        ClearCollect(
            colRefActiveIds,
            AddColumns(
                ShowColumns(
                    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                    "CatItem_ID"
                ),
                id, Text(CatItem_ID)
            )
        );

        // 2) base cat items for this category (with ids)
        ClearCollect(
            colCatBase,
            AddColumns(
                Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                id, Text(CatItem_ID)
            )
        );

        // 3) desired = overlay toggles on current check-state
        ClearCollect(
            colDesired,
            AddColumns(
                AddColumns(
                    colCatBase,
                    curSelected, CountIf(colRefActiveIds, id = ThisRecord.id) > 0
                ),
                Desired,
                    With(
                        { tgl: LookUp(colToggleLog, id = id) },
                        If(IsBlank(tgl), curSelected, tgl.Desired)
                    )
            )
        );

        // 4) diffs
        ClearCollect(colAdds,    Filter(colDesired, Desired = true  && curSelected = false));
        ClearCollect(colRemoves, Filter(colDesired, Desired = false && curSelected = true));

        // 5) APPLY ADDS: reactivate if exists else create
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
                    Patch(
                        Skill_Matrix_Reference,
                        existingRef,
                        { IsActive: true, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                    ),
                    With(
                        { newRefId: GUID() },
                        Patch(
                            Skill_Matrix_Reference,
                            Defaults(Skill_Matrix_Reference),
                            {
                                Reference_ID: newRefId,
                                Mod_ID: varSelectedModuleId,
                                Module: varSelectedModuleName,
                                CatItem_ID: a.CatItem_ID,
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

        // 6) APPLY REMOVES (soft delete)
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
                    Patch(
                        Skill_Matrix_Reference,
                        activeRef,
                        { IsActive: false, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                    )
                )
            )
        );

        // 7) Skill_Type edits (CategoryItems only; keep or drop as you need)
        ClearCollect(
            colCI_Changed,
            Filter(
                AddColumns(
                    galCatItems.AllItems,
                    NewType, Coalesce(drpCIType.Selected.Value, Skill_Type),
                    OldType, Skill_Type
                ),
                NewType <> OldType
            )
        );
        ForAll(
            colCI_Changed As c,
            Patch(
                Skill_Matrix_CategoryItems,
                LookUp(Skill_Matrix_CategoryItems, CatItem_ID = c.CatItem_ID),
                { Skill_Type: c.NewType, Modified_By: reqUserEmail, Modified_At: Now() }
            )
        );

        // 8) counts + assignments (optional — keep if you need the flow)
        Set(varAddCount,    CountRows(colAdds));
        Set(varRemoveCount, CountRows(colRemoves));
        Set(varUpdCount,    CountRows(colCI_Changed));
        Set(varCodeNow,     "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

        If(
            varAddCount + varRemoveCount > 0,
            With(
                {
                    addedRefsText:
                        Concat(
                            ForAll(
                                colAdds As a2,
                                With(
                                    { rr: LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a2.id) },
                                    If(!IsBlank(rr), rr.Reference_ID)
                                )
                            ),
                            Value,
                            ";"
                        ),
                    removedRefsText:
                        Concat(
                            ForAll(
                                colRemoves As r2,
                                With(
                                    { rr: LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = r2.id) },
                                    If(!IsBlank(rr), rr.Reference_ID)
                                )
                            ),
                            Value,
                            ";"
                        )
                },
                Patch(
                    Skill_Matrix_Assignments,
                    Defaults(Skill_Matrix_Assignments),
                    {
                        Title: varCodeNow,
                        Operation: { Value: If(varAddCount>0 && varRemoveCount>0, "ModCatItemAddRem", If(varAddCount>0, "ModCatItemAdded", "ModCatItemRemoved")) },
                        Status: { Value: "Pending" },
                        Module_ID: varSelectedModuleId,
                        Module_Name: varSelectedModuleName,
                        Added_Reference_IDs: addedRefsText,
                        Removed_Reference_IDs: removedRefsText,
                        Added_Count: varAddCount,
                        Removed_Count: varRemoveCount,
                        RequestedAt: Now()
                    }
                )
            )
        );

        If(
            varUpdCount > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: { Value: "CatItem_Updated" },
                    Status: { Value: "Pending" },
                    Module_ID: varSelectedModuleId,
                    Module_Name: varSelectedModuleName,
                    Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
                    Updated_Count: varUpdCount,
                    RequestedAt: Now()
                }
            )
        );

        // 9) Notify
        If(
            varAddCount + varRemoveCount + varUpdCount > 0,
            Notify(
                "Saved. Added: " & varAddCount & " | Removed: " & varRemoveCount & " | Skill-type updates: " & varUpdCount,
                NotificationType.Success
            ),
            Notify("No changes to save.", NotificationType.Information)
        );

        // 10) Authoritative rebuild from server
        Refresh(Skill_Matrix_Reference);
        Clear(colToggleLog);

        ClearCollect(
            colRefActiveIds,
            AddColumns(
                ShowColumns(
                    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                    "CatItem_ID"
                ),
                id, Text(CatItem_ID)
            )
        );

        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                AddColumns(
                    Filter(
                        Skill_Matrix_CategoryItems,
                        Text(Category) = catText
                    ),
                    id, Text(CatItem_ID)
                ),
                IsSelectedForModule,
                CountIf(colRefActiveIds, id = ThisRecord.id) > 0
            )
        )
    )
)





"active=" & CountRows(colRefActiveIds) &
" | cat=" & varSelectedCategoryText &
" | toggles=" & CountRows(colToggleLog)




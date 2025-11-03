// cache selection
Set(varSelectedModuleId, cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// build active refs for this module as TEXT ids
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

// clear toggles when switching modules (optional but safer)
Clear(colToggleLog);




// normalize cat text (works for Choice/Text)
Set(varSelectedCategoryText, If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)));

// sanity: refresh active list for safety if needed
If(!IsBlank(varSelectedModuleId),
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

// clear toggles when switching categories (optional)
Clear(colToggleLog);









With(
    { catText: varSelectedCategoryText },
    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        FirstN(Skill_Matrix_CategoryItems, 0),
        AddColumns(
            AddColumns(
                Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                id, Text(CatItem_ID)
            ),
            // what is active today for this module?
            IsSelectedForModule,
            CountIf(colRefActiveIds, id = ThisRecord.id) > 0
        )
    )
)



// ======================= Save Module Config (deterministic) =======================

// 0) Who + guards
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

If(
    IsBlank(varSelectedModuleId) || IsBlank(varSelectedCategoryText),
    Notify("Pick a module and a category first.", NotificationType.Warning),
    
    With(
        { catText: varSelectedCategoryText },

        // 1) Snapshot: ACTIVE refs for this module (TEXT ids)
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

        // 2) Category’s CatItems (normalized id + curSelected)
        ClearCollect(
            colCatBase,
            AddColumns(
                Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                id, Text(CatItem_ID),
                curSelected, CountIf(colRefActiveIds, id = ThisRecord.id) > 0
            )
        );

        // 3) Desired overlay = current OR user toggle if present
        ClearCollect(
            colDesired,
            AddColumns(
                colCatBase,
                Desired,
                    With(
                        { tgl: LookUp(colToggleLog, CatItem_ID = id) },
                        If(IsBlank(tgl), curSelected, tgl.Desired)
                    )
            )
        );

        // 4) Diffs
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
                                CatItem_ID: a.CatItem_ID,      // CatItem_ID is TEXT in SP; if you see a type error, use: a.id
                                Category: a.Category,
                                Item: a.Item,
                                SkillLevel: a.Skill_Type,       // (internal name may be SkillLevel)
                                IsActive: true,
                                Created_By: reqUserEmail,
                                Created_At: Now()
                            }
                        )
                    )
                )
            )
        );

        // 6) APPLY REMOVES (soft)
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

        // 7) Skill_Type edits (CategoryItems only; optional)
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

        // 8) Counts
        Set(varAddCount,     CountRows(colAdds));
        Set(varRemoveCount,  CountRows(colRemoves));
        Set(varUpdCount,     CountRows(colCI_Changed));
        Set(varCodeNow,      "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

        // 9) Assignment logging (Module deltas)
        If(
            varAddCount + varRemoveCount > 0,
            With(
                {
                    addedRefsText:
                        Concat(
                            ForAll(
                                colAdds As a2,
                                // try to read the (re)activated/created Reference_ID
                                Coalesce(
                                    LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a2.id, Reference_ID),
                                    ""      // fallback empty if immediate read hasn’t propagated
                                )
                            ),
                            Value, ";"
                        ),
                    removedRefsText:
                        Concat(
                            ForAll(
                                colRemoves As r2,
                                LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = r2.id, Reference_ID)
                            ),
                            Value, ";"
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
                        Added_Reference_IDs:   addedRefsText,
                        Removed_Reference_IDs: removedRefsText,
                        Added_Count: varAddCount,
                        Removed_Count: varRemoveCount,
                        RequestedAt: Now()
                    }
                )
            )
        );

        // 10) Assignment logging (CatItem type updates)
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

        // 11) Notify
        If(
            (varAddCount + varRemoveCount + varUpdCount) > 0,
            Notify("Saved. Added: " & varAddCount & " | Removed: " & varRemoveCount & " | Skill-type updates: " & varUpdCount, NotificationType.Success),
            Notify("No changes to save.", NotificationType.Information)
        );

        // 12) Hard refresh + rebuild view (authoritative)
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
        )
        // (Gallery will recompute IsSelectedForModule from colRefActiveIds automatically)
    )
);


"active=" & CountRows(colRefActiveIds) &
" | cat=" & varSelectedCategoryText &
" | toggles=" & CountRows(colToggleLog)





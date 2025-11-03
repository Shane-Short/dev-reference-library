/* ---------- 5) DELTA: apply adds/removes for the CURRENT category only ---------- */

With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },

    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        Notify("Pick a module and a category.", NotificationType.Warning),
        
        With(
            {
                /* 5.1 Active references for this module (normalize id to text) */
                curActive:
                    AddColumns(
                        ShowColumns(
                            Filter(
                                Skill_Matrix_Reference,
                                Mod_ID = varSelectedModuleId && IsActive = true
                            ),
                            "CatItem_ID"
                        ),
                        "id", Text(CatItem_ID)         // <-- if numeric ID, use Value(CatItem_ID)
                    ),

                /* 5.2 Desired selection from the working list (only this category) */
                desiredSel:
                    AddColumns(
                        ShowColumns(
                            Filter(
                                colModuleCatItems_Working,
                                IsSelectedForModule = true
                            ),
                            "id"
                        ),
                        "id", Text(id)                 // normalize (id already text, keep for symmetry)
                    ),

                /* 5.3 All CatItems currently visible (this category) as ids */
                thisCatIds:
                    AddColumns(
                        ShowColumns(
                            AddColumns(
                                Filter(
                                    Skill_Matrix_CategoryItems,
                                    Text(Category) = catText
                                ),
                                "id", Text(CatItem_ID)
                            ),
                            "id"
                        ),
                        "id", id
                    )
            },

            /* 5.4 Compute adds: chosen now but not previously active */
            ClearCollect(
                _adds,
                Filter(desiredSel As d, CountIf(curActive, id = d.id) = 0)
            );

            /* 5.5 Compute removes: were active + in this category, but no longer selected */
            ClearCollect(
                _removes,
                Filter(
                    curActive As c,
                    CountIf(thisCatIds, id = c.id) > 0    /* only items from THIS category */
                    && CountIf(desiredSel, id = c.id) = 0 /* now unselected */
                )
            );

            /* 5.6 APPLY ADDS */
            ForAll(
                _adds As a,
                With(
                    {
                        ci: LookUp(Skill_Matrix_CategoryItems, Text(CatItem_ID) = a.id) /* if numeric: Value(CatItem_ID) = Value(a.id) */,
                        refRow: LookUp(
                                    Skill_Matrix_Reference,
                                    Mod_ID = varSelectedModuleId
                                    && Text(CatItem_ID) = a.id                          /* if numeric: CatItem_ID = Value(a.id) */
                                )
                    },
                    If(
                        /* re-activate existing ref if found */
                        !IsBlank(refRow),
                        Patch(
                            Skill_Matrix_Reference,
                            refRow,
                            { IsActive: true, Updated_By: User().Email, Updated_At: Now() }
                        ),
                        /* else create a new ref row */
                        With(
                            { newRefId: GUID() },
                            Patch(
                                Skill_Matrix_Reference,
                                Defaults(Skill_Matrix_Reference),
                                {
                                    Reference_ID: newRefId,
                                    Mod_ID: varSelectedModuleId,
                                    Module: varSelectedModuleName,

                                    CatItem_ID: ci.CatItem_ID,
                                    Category: ci.Category,
                                    Item: ci.Item,

                                    /* in Reference this column is “SkillLevel” (display “Skill_Type”) */
                                    SkillLevel: ci.Skill_Type,

                                    IsActive: true,
                                    Created_By: User().Email,
                                    Created_At: Now()
                                }
                            )
                        )
                    )
                )
            );

            /* 5.7 APPLY REMOVES (soft delete) */
            ForAll(
                _removes As r,
                With(
                    {
                        refRow: LookUp(
                                    Skill_Matrix_Reference,
                                    Mod_ID = varSelectedModuleId
                                    && Text(CatItem_ID) = r.id                        /* if numeric: CatItem_ID = Value(r.id) */
                                )
                    },
                    If(
                        !IsBlank(refRow),
                        Patch(
                            Skill_Matrix_Reference,
                            refRow,
                            { IsActive: false, Updated_By: User().Email, Updated_At: Now() }
                        )
                    )
                )
            );

            /* 5.8 Accurate counts + notify */
            Set(varAddCount, CountRows(_adds));
            Set(varRemoveCount, CountRows(_removes));

            If(
                varAddCount + varRemoveCount = 0,
                Notify("No changes to save.", NotificationType.Information),
                Notify(
                    "Saved. Added " & Text(varAddCount) & " | Removed " & Text(varRemoveCount) & ".",
                    NotificationType.Success
                )
            );

            /* 5.9 REBUILD working set so the checks reflect the Reference table immediately */
            /* (Use the same REBUILD block we just fixed earlier — alias-safe version) */
            /* --- Active refs snapshot --- */
            ClearCollect(
                colRefActiveIds,
                AddColumns(
                    ShowColumns(
                        Filter(
                            Skill_Matrix_Reference,
                            Mod_ID = varSelectedModuleId && IsActive = true
                        ),
                        "CatItem_ID"
                    ),
                    "id", Text(CatItem_ID)
                )
            );
            /* --- Category list with IsSelectedForModule --- */
            With(
                { _catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
                If(
                    IsBlank(varSelectedModuleId) || IsBlank(_catText),
                    Clear(colModuleCatItems_Working),
                    With(
                        {
                            catList:
                                AddColumns(
                                    Filter(
                                        Skill_Matrix_CategoryItems,
                                        Text(Category) = _catText
                                    ),
                                    "id", Text(CatItem_ID)
                                )
                        },
                        ClearCollect(
                            colModuleCatItems_Working,
                            AddColumns(
                                catList,
                                "IsSelectedForModule",
                                CountIf(colRefActiveIds As r, r.id = id) > 0
                            )
                        )
                    )
                )
            )
        )
    )
);

/* optional: clean temp */
Clear(_adds); Clear(_removes);

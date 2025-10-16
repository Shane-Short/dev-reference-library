// ===================== Save Module (New or Existing) =====================

// 0) Validate: must have an existing module selected OR a new module name typed
If(
    IsBlank(varSelectedModuleId) && IsBlank(Trim(txtNewModuleName.Text)),
    Notify("Pick an existing Module or enter a new Module name.", NotificationType.Warning),

    With(
        {
            // Resolve module identity for both branches
            moduleId: If(
                !IsBlank(varSelectedModuleId),
                varSelectedModuleId,
                GUID()                               // new module id if creating
            ),
            moduleName: If(
                !IsBlank(varSelectedModuleId),
                varSelectedModuleName,
                Proper(Trim(txtNewModuleName.Text))   // normalized new module name
            )
        },

        // 1) If NEW module → insert to Modules
        If(
            IsBlank(varSelectedModuleId),
            Patch(
                Skill_Matrix_Modules,
                Defaults(Skill_Matrix_Modules),
                {
                    Module_ID:  moduleId,
                    Title:      moduleName,
                    Created_By: User().Email,
                    Created_At: Now()
                }
            )
        );

        // 2) Persist Skill_Type edits back to CategoryItems (source of truth)
        ForAll(
            colSkillTypeEdits As e,
            Patch(
                Skill_Matrix_CategoryItems,
                LookUp(Skill_Matrix_CategoryItems, CatItem_ID = e.CatItem_ID),
                {
                    Skill_Type:  e.Skill_Type,
                    Modified_By: User().Email,
                    Modified_At: Now()
                }
            )
        );

        // 3) Compute diffs: what to ADD and what to REMOVE
        ClearCollect(
            colToAdd,
            Filter(
                colModuleCatItems_Working As w,
                IsBlank(
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = moduleId && CatItem_ID = w.CatItem_ID
                    )
                )
            )
        );

        ClearCollect(
            colToRemove,
            Filter(
                Skill_Matrix_Reference As r,
                Mod_ID = moduleId &&
                IsBlank(
                    LookUp(colModuleCatItems_Working, CatItem_ID = r.CatItem_ID)
                )
            )
        );

        // 4) ADD new Reference links (with denormalized fields)
        Clear(colDidAdd);
        ForAll(
            colToAdd As a,
            With(
                { ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = a.CatItem_ID) },
                Patch(
                    Skill_Matrix_Reference,
                    Defaults(Skill_Matrix_Reference),
                    {
                        Reference_ID: GUID(),
                        Mod_ID:       moduleId,
                        CatItem_ID:   a.CatItem_ID,

                        // Denormalized (for current UX/flows)
                        Category:     ci.Category,
                        Item:         ci.Item,
                        Module:       moduleName,
                        Skill_Type:   ci.Skill_Type,

                        Created_By:   User().Email,
                        Created_At:   Now()
                    }
                );
                Collect(colDidAdd, { id: a.CatItem_ID })
            )
        );

        // 5) REMOVE Reference links that were unchecked
        Clear(colDidRemove);
        ForAll(
            colToRemove As r,
            Remove(Skill_Matrix_Reference, r);
            Collect(colDidRemove, { id: r.CatItem_ID })
        );

        // 6) OPTIONAL: Keep existing Reference rows' Skill_Type synced with CategoryItems
        Clear(colDidUpdate);
        ForAll(
            Filter(Skill_Matrix_Reference As r, Mod_ID = moduleId),
            With(
                { ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = r.CatItem_ID) },
                If(
                    !IsBlank(ci) && ci.Skill_Type <> r.Skill_Type,
                    Patch(Skill_Matrix_Reference, r, { Skill_Type: ci.Skill_Type });
                    Collect(colDidUpdate, { id: r.CatItem_ID })
                )
            )
        );

        // 7) Notify with counts (handle no-op gracefully)
        If(
            CountRows(colDidAdd) + CountRows(colDidRemove) + CountRows(colDidUpdate) = 0,
            Notify("No changes to save.", NotificationType.Information),
            Notify(
                "Saved. Added: " & CountRows(colDidAdd) &
                " | Removed: " & CountRows(colDidRemove) &
                " | Updated types: " & CountRows(colDidUpdate),
                NotificationType.Success
            )
        );

        // 8) Cleanup & UI reset
        Clear(colModuleCatItems_Working);
        Clear(colSkillTypeEdits);
        Clear(colToAdd);
        Clear(colToRemove);
        Clear(colDidAdd);
        Clear(colDidRemove);
        Clear(colDidUpdate);

        Reset(txtNewModuleName);
        Reset(cmbExistingModule);
        Reset(txtNewCategory);
        Reset(cmbSelectCategory);
        Reset(txtBulkItems);

        Set(varSelectedModuleId, Blank());
        Set(varSelectedModuleName, "");

        // Optional refresh if you want immediate re-query from SharePoint
        // Refresh(Skill_Matrix_Reference);
        // Refresh(Skill_Matrix_CategoryItems)
    )
)

// ---------- 0) BEFORE snapshot of Reference rows for this module ----------
ClearCollect(
    colRefBeforeSave,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
        "Reference_ID", "CatItem_ID", "Mod_ID"
    )
);

// ---------- 0a) Validate ----------
If(
    IsBlank(varSelectedModuleId) && IsBlank(Trim(varSelectedModuleName)),
    Notify("Select an existing Module or enter a new Module name.", NotificationType.Warning),
    
    With(
        {
            // Resolve module identity for both paths
            moduleId: If(IsBlank(varSelectedModuleId), GUID(), varSelectedModuleId),
            moduleName: If(IsBlank(varSelectedModuleId), Proper(Trim(varSelectedModuleName)), varSelectedModuleName)
        },

        // ---------- 1) If NEW module, insert it ----------
        If(
            IsBlank(varSelectedModuleId),
            Patch(
                Skill_Matrix_Modules,
                Defaults(Skill_Matrix_Modules),
                { Mod_ID: moduleId, Title: moduleName, Created_By: User().Email, Created_At: Now() }
            )
        );

        // ---------- 2) Write Skill_Type edits back to CategoryItems (source of truth) ----------
        ForAll(
            colSkillTypeEdits As e,
            Patch(
                Skill_Matrix_CategoryItems,
                LookUp(Skill_Matrix_CategoryItems, CatItem_ID = e.CatItem_ID),
                {
                    Skill_Type: e.Skill_Type,
                    Created_By: User().Email,
                    Created_At: Now()
                }
            )
        );

        // ---------- 2b) Build CatItem universe for skill-type diff (module OR category-only) ----------
        ClearCollect(
            colCIUniverse,
            If(
                !IsEmpty(colModuleCatItems_Working),
                ShowColumns(colModuleCatItems_Working, "CatItem_ID"),
                ShowColumns(
                    If(
                        !IsBlank(drpCategory.Selected.Value),
                        Filter(Skill_Matrix_CategoryItems, Category = drpCategory.Selected.Value),
                        Skill_Matrix_CategoryItems
                    ),
                    "CatItem_ID"
                )
            )
        );
        ClearCollect(colCIIds, Distinct(colCIUniverse, CatItem_ID));

        // BEFORE snapshot of CategoryItems' Skill_Type for those IDs
        ClearCollect(
            colCIBefore,
            ForAll(
                colCIIds As t,
                {
                    CatItem_ID: Coalesce(t.Result, t.Value),
                    Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = Coalesce(t.Result, t.Value), Skill_Type)
                }
            )
        );

        // ---------- 3) Compute ADD/REMOVE vs Reference for this module ----------
        ClearCollect(
            colToAdd,
            Filter(
                colModuleCatItems_Working As w,
                w.IsChecked = true &&
                IsBlank(
                    LookUp(Skill_Matrix_Reference, Mod_ID = moduleId && CatItem_ID = w.CatItem_ID)
                )
            )
        );

        ClearCollect(
            colToRemove,
            Filter(
                Skill_Matrix_Reference As r,
                r.Mod_ID = moduleId &&
                IsBlank(
                    LookUp(colModuleCatItems_Working, IsChecked = true && CatItem_ID = r.CatItem_ID)
                )
            )
        );

        // ---------- 4) ADD Reference links ----------
        Clear(colDidAdd);
        ForAll(
            colToAdd As a,
            With(
                { ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = a.CatItem_ID) },
                If(
                    !IsBlank(ci),
                    Patch(
                        Skill_Matrix_Reference,
                        Defaults(Skill_Matrix_Reference),
                        {
                            Reference_ID: GUID(),
                            Mod_ID: moduleId,
                            CatItem_ID: a.CatItem_ID,

                            // denormalized
                            Module: moduleName,
                            Category: ci.Category,
                            Item: ci.Item,
                            SkillLevel: ci.Skill_Type,      // <- Reference list uses SkillLevel

                            Created_By: User().Email,
                            Created_At: Now()
                        }
                    );
                    Collect(colDidAdd, { CatItem_ID: a.CatItem_ID })
                )
            )
        );

        // ---------- 5) REMOVE Reference links (batch off the DS) ----------
        ClearCollect(
            colRefsToDelete,
            ShowColumns(colToRemove, "Reference_ID")
        );
        If(
            CountRows(colRefsToDelete) > 0,
            RemoveIf(Skill_Matrix_Reference, Reference_ID in colRefsToDelete.Reference_ID)
        );
        Clear(colDidRemove);
        ForAll(colToRemove As r, Collect(colDidRemove, { CatItem_ID: r.CatItem_ID }));

        // ---------- 6) OPTIONAL: sync existing Reference rows' SkillLevel with CategoryItems ----------
        ClearCollect(
            colRefsForSync,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = moduleId),
                "Reference_ID", "CatItem_ID", "SkillLevel"
            )
        );
        Clear(colDidUpdate);
        ForAll(
            colRefsForSync As ref,
            With(
                { ci2: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ref.CatItem_ID) },
                If(
                    !IsBlank(ci2) && !IsBlank(ci2.Skill_Type) && ci2.Skill_Type <> ref.SkillLevel,
                    Patch(
                        Skill_Matrix_Reference,
                        LookUp(Skill_Matrix_Reference, Reference_ID = ref.Reference_ID),
                        { SkillLevel: ci2.Skill_Type }
                    );
                    Collect(colDidUpdate, { CatItem_ID: ref.CatItem_ID })
                )
            )
        );

        // ---------- 7) AFTER snapshot (Reference) + diffs by Reference_ID ----------
        ClearCollect(
            colRefAfterSave,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = moduleId),
                "Reference_ID", "CatItem_ID", "Mod_ID"
            )
        );
        ClearCollect(
            colAddedRefs,
            Filter(colRefAfterSave As a, CountIf(colRefBeforeSave, Reference_ID = a.Reference_ID) = 0)
        );
        ClearCollect(
            colRemovedRefs,
            Filter(colRefBeforeSave As b, CountIf(colRefAfterSave, Reference_ID = b.Reference_ID) = 0)
        );

        // counts + operation name
        Set(addCnt, CountRows(colAddedRefs));
        Set(remCnt, CountRows(colRemovedRefs));
        Set(
            opValue,
            If(
                addCnt > 0 && remCnt > 0, "ModCatItemAddRem",
                addCnt > 0,               "ModCatItemAdded",
                remCnt > 0,               "ModCatItemRemoved",
                                          "ModuleUpdated"
            )
        );

        // Resolve module id text safely for Assignments
        Set(varModuleIdSafe, "" & moduleId);

        // ---------- 8) QUEUE ModuleUpdated-type assignment ----------
        With(
            {
                code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                req:  Office365Users.UserProfileV2(User().Email)
            },
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: code,
                    Operation: { Value: opValue },
                    Status:    { Value: "Pending" },

                    Module_ID:   varModuleIdSafe,
                    Module_Name: moduleName,

                    // Reference_ID payloads (preferred)
                    Added_Reference_IDs:   Concat(colAddedRefs, Reference_ID & ";"),
                    Removed_Reference_IDs: Concat(colRemovedRefs, Reference_ID & ";"),

                    // Legacy CatItem_ID payloads (keep for now)
                    Added_CatItem_IDs:   Concat(colDidAdd, CatItem_ID & ";"),
                    Removed_CatItem_IDs: Concat(colDidRemove, CatItem_ID & ";"),

                    Added_Count:  addCnt,
                    Removed_Count: remCnt,
                    Updated_Count: CountRows(colDidUpdate),

                    RequestedAt: Now(),
                    RequestedBy: {
                        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                        Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
                        DisplayName: Coalesce(req.displayName, User().FullName),
                        Email: Coalesce(req.mail, User().Email),
                        Department: Coalesce(req.department, ""),
                        JobTitle: Coalesce(req.jobTitle, ""),
                        Picture: ""
                    }
                }
            )
        );

        // ---------- 9) CATEGORY-ONLY Skill_Type change detection + queue EditCI ----------
        // AFTER snapshot of CategoryItems for the same CatItem universe
        ClearCollect(
            colCIAfter,
            ForAll(
                colCIIds As t,
                {
                    CatItem_ID: Coalesce(t.Result, t.Value),
                    Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = Coalesce(t.Result, t.Value), Skill_Type)
                }
            )
        );

        Clear(colCIChanged);
        ForAll(
            colCIAfter As a,
            If(
                LookUp(colCIBefore, CatItem_ID = a.CatItem_ID, Skill_Type) <> a.Skill_Type,
                Collect(colCIChanged, { CatItem_ID: a.CatItem_ID })
            )
        );
        ClearCollect(colCIChanged, Distinct(colCIChanged, CatItem_ID));

        If(
            CountRows(colCIChanged) > 0,
            With(
                {
                    code2: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                    req2:  Office365Users.UserProfileV2(User().Email)
                },
                Patch(
                    Skill_Matrix_Assignments,
                    Defaults(Skill_Matrix_Assignments),
                    {
                        Title: code2,
                        Operation: { Value: "CatItem_Update" },
                        Status:    { Value: "Pending" },
                        Updated_CatItem_IDs: Concat(colCIChanged, Text(Result) & ";"),
                        Updated_Count:       CountRows(colCIChanged),

                        RequestedAt: Now(),
                        RequestedBy: {
                            '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                            Claims: "i:0#.f|membership|" & Coalesce(req2.mail, User().Email),
                            DisplayName: Coalesce(req2.displayName, User().FullName),
                            Email: Coalesce(req2.mail, User().Email),
                            Department: Coalesce(req2.department, ""),
                            JobTitle: Coalesce(req2.jobTitle, ""),
                            Picture: ""
                        }
                    }
                )
            )
        );

        // ---------- 10) Notify ----------
        If(
            addCnt + remCnt + CountRows(colCIChanged) = 0,
            Notify("No changes to save.", NotificationType.Information),
            Notify(
                "Saved. Added: " & addCnt & " | Removed: " & remCnt & " | Skill type updates: " & CountRows(colCIChanged) & ".",
                NotificationType.Success
            )
        );

        // ---------- 11) Refresh + cleanup ----------
        Refresh(Skill_Matrix_Reference);
        Refresh(Skill_Matrix_CategoryItems);
        Refresh(Skill_Matrix_Assignments);

        Clear(colRefBeforeSave);
        Clear(colRefAfterSave);
        Clear(colToAdd);
        Clear(colToRemove);
        Clear(colRefsToDelete);
        Clear(colRefsForSync);
        Clear(colDidAdd);
        Clear(colDidRemove);
        Clear(colDidUpdate);

        Clear(colCIUniverse);
        Clear(colCIIds);
        Clear(colCIBefore);
        Clear(colCIAfter);
        Clear(colCIChanged);

        // (optional) reset controls as you prefer
        // Reset(cmbEditModules); Set(varSelectedModuleId, Blank()); Set(varSelectedModuleName, Blank())
    )
);

// 2) Build proposed CatItem type states from the gallery safely
Clear(colCI_Proposed);

ClearCollect(
    colCI_Proposed,
    ForAll(
        galCatItems.AllItems As row,
        With(
            {
                // Always pull the persisted type from the list, not the UI
                oldTypeText:
                    Text(
                        LookUp(
                            Skill_Matrix_CategoryItems,
                            CatItem_ID = row.CatItem_ID,
                            Skill_Type
                        )
                    ),

                // Coerce whatever the UI control returns into TEXT
                pickedTypeText:
                    Coalesce(
                        // Dropdown inside the gallery (preferred)
                        Text(row.drpCIType.Selected.Value),
                        // If you happen to use a ComboBox instead (single-select)
                        If(
                            !IsBlank(row.cmbCIType.SelectedItems),
                            Text(First(row.cmbCIType.SelectedItems).Value)
                        ),
                        // Or a text input (last resort)
                        Text(Trim(row.txtCIType.Text))
                    )
            },
            {
                CatItem_ID: row.CatItem_ID,
                OldType: oldTypeText,
                NewType: If(IsBlank(pickedTypeText), oldTypeText, pickedTypeText)
            }
        )
    )
);

// 2.1 Compute only the rows that actually changed (keep if already present)

Clear(colCI_Updates);
ClearCollect(
    colCI_Updates,
    Filter(
        colCI_Proposed,
        !IsBlank(CatItem_ID) && NewType <> OldType
    )
);

// 2.2 Persist the changed Skill_Type back to CategoryItems
ForAll(
    colCI_Updates As u,
    Patch(
        Skill_Matrix_CategoryItems,
        LookUp(Skill_Matrix_CategoryItems, CatItem_ID = u.CatItem_ID),
        {
            Skill_Type: u.NewType,
            Modified_By: User().Email,
            Modified_At: Now()
        }
    )
);


2.3) Queue the EditCI assignment (IDs payload)

If(
    CountRows(colCI_Updates) > 0,
    With(
        {
            code: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
            req:  Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: code,
                Operation: { Value: "CatItem_Updated" },
                Status:    { Value: "Pending" },

                // Semicolon-delimited IDs of changed CatItems
                Updated_CatItem_IDs: Concat(colCI_Updates, CatItem_ID, ";"),
                Updated_Count:       CountRows(colCI_Updates),

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
    )
);

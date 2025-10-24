// Nothing loaded?
If(
    IsEmpty(colUserSelections),
    Notify("Nothing to save yet.", NotificationType.Information),
    
    With(
        { changed: Filter(colUserSelections, HasChanged = true) },
        
        // No pending changes?
        If(
            CountRows(changed) = 0,
            Notify("No changes to save.", NotificationType.Information),
            
            // Try to upsert each changed row
            ForAll(
                changed As row,
                With(
                    {
                        // Reference row for this Module/Category/Item (may be blank if missing)
                        ref: LookUp(
                                Skill_Matrix_Reference,
                                Module = row.Module && Category = row.Category && Item = row.Item
                             ),
                        // Common fields we always send (match your SharePoint schema names)
                        base:
                            {
                                // Many SharePoint lists require Title; give a deterministic one:
                                Title: row.Module & " | " & row.Category & " | " & row.Item,

                                Employee_Email: varMe,
                                Module: row.Module,
                                Category: row.Category,
                                Item: row.Item,

                                // Defaults if blank
                                Skill_Level: Value(Coalesce(row.Skill_Level, 1)),
                                Skill_Type: Coalesce(ref.Skill_Type, "Basic"),

                                // You switched these to Single line of text → safe to write GUID strings
                                Reference_ID: ref.Reference_ID,
                                Mod_ID: ref.Mod_ID,

                                Timestamp: Now()
                            }
                    },

                    // First, attempt as if Employee is a PERSON column
                    IfError(
                        Patch(
                            Skill_Matrix_Entries,
                            LookUp(
                                Skill_Matrix_Entries,
                                Lower(Trim(Employee_Email)) = varMe &&
                                Module   = row.Module &&
                                Category = row.Category &&
                                Item     = row.Item
                            ),
                            base & {
                                Employee: {
                                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                                    Claims: "i:0#.f|membership|" & varMe,
                                    DisplayName: Coalesce(Office365Users.MyProfileV2().displayName, User().FullName),
                                    Email: varMe,
                                    Department: "",
                                    JobTitle: "",
                                    Picture: ""
                                }
                            }
                        ),
                        // If that fails (e.g., Employee is TEXT), try again with plain text
                        Patch(
                            Skill_Matrix_Entries,
                            LookUp(
                                Skill_Matrix_Entries,
                                Lower(Trim(Employee_Email)) = varMe &&
                                Module   = row.Module &&
                                Category = row.Category &&
                                Item     = row.Item
                            ),
                            base & {
                                Employee: Coalesce(Office365Users.MyProfileV2().displayName, User().FullName)
                            }
                        )
                    )
                )
            );

            // Mark local rows clean
            UpdateIf(colUserSelections, HasChanged = true, { HasChanged: false });

            // Optional: inspect SharePoint write errors if anything still fails silently
            // ClearCollect(colEntryErrors, Errors(Skill_Matrix_Entries));
            // If(CountRows(colEntryErrors)>0, Notify("Some rows failed to save. Check colEntryErrors.", NotificationType.Error));

            Notify("Saved " & Text(CountRows(changed)) & " change(s).", NotificationType.Success)
        )
    )
)

// Guard: nothing loaded
If(
    IsEmpty(colUserSelections),
    Notify("Nothing to save yet.", NotificationType.Information),
    
    With(
        { changed: Filter(colUserSelections, HasChanged = true) },

        // Guard: no changes
        If(
            CountRows(changed) = 0,
            Notify("No changes to save.", NotificationType.Information),

            // Upsert each changed row
            ForAll(
                changed As row,
                With(
                    {
                        // Pull reference fields individually to avoid '.' on blank/error record
                        refSkillType: LookUp(
                            Skill_Matrix_Reference,
                            Module = row.Module && Category = row.Category && Item = row.Item,
                            Skill_Type
                        ),
                        refRefId: LookUp(
                            Skill_Matrix_Reference,
                            Module = row.Module && Category = row.Category && Item = row.Item,
                            Reference_ID
                        ),
                        refModId: LookUp(
                            Skill_Matrix_Reference,
                            Module = row.Module && Category = row.Category && Item = row.Item,
                            Mod_ID
                        )
                    },
                    Patch(
                        Skill_Matrix_Entries,
                        // Update existing row for this user+M/C/I, or create if not found
                        LookUp(
                            Skill_Matrix_Entries,
                            Lower(Trim(Employee_Email)) = varMe &&
                            Module   = row.Module &&
                            Category = row.Category &&
                            Item     = row.Item
                        ),
                        {
                            // Many SharePoint lists have Title required
                            Title: row.Module & " | " & row.Category & " | " & row.Item,

                            // Text columns (per your schema)
                            Employee:       User().FullName,
                            Employee_Email: varMe,

                            Module:   row.Module,
                            Category: row.Category,
                            Item:     row.Item,

                            // Defaults if blank
                            Skill_Level: Value(Coalesce(row.Skill_Level, 1)),
                            Skill_Type:  Coalesce(refSkillType, "Basic"),

                            // GUIDs stored as text in Entries (you changed these columns to text)
                            Reference_ID: refRefId,
                            Mod_ID:       refModId,

                            Timestamp: Now()
                        }
                    )
                )
            );

            // Mark local rows clean
            UpdateIf(colUserSelections, HasChanged = true, { HasChanged: false });

            // Optional: capture SharePoint write errors for debugging
            // ClearCollect(colEntryErrors, Errors(Skill_Matrix_Entries));
            // If(CountRows(colEntryErrors) > 0, Notify("Some rows failed to save. Check colEntryErrors.", NotificationType.Error));

            Notify("Saved " & Text(CountRows(changed)) & " change(s).", NotificationType.Success)
        )
    )
)

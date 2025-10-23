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
            
            // Else: upsert each changed row
            ForAll(
                changed As row,
                Patch(
                    Skill_Matrix_Entries,
                    LookUp(
                        Skill_Matrix_Entries,
                        Lower(Trim(Employee_Email)) = varMe &&
                        Module = row.Module &&
                        Category = row.Category &&
                        Item = row.Item
                    ),
                    {
                        Employee: Coalesce(Office365Users.MyProfileV2().displayName, User().FullName),
                        Employee_Email: varMe,
                        Module: row.Module,
                        Category: row.Category,
                        Item: row.Item,
                        Skill_Level: Value(row.Skill_Level),
                        Skill_Type: LookUp(
                            Skill_Matrix_Reference,
                            Module = row.Module && Category = row.Category && Item = row.Item,
                            Skill_Type
                        ),
                        // --- OPTIONAL: uncomment if these columns exist in Skill_Matrix_Entries ---
                        // Reference_ID: LookUp(
                        //     Skill_Matrix_Reference,
                        //     Module = row.Module && Category = row.Category && Item = row.Item,
                        //     Reference_ID
                        // ),
                        // Mod_ID: LookUp(
                        //     Skill_Matrix_Reference,
                        //     Module = row.Module && Category = row.Category && Item = row.Item,
                        //     Mod_ID
                        // ),
                        Timestamp: Now()
                    }
                )
            );
            
            // Mark local rows as clean (don’t rebuild the whole collection)
            UpdateIf(colUserSelections, HasChanged = true, { HasChanged: false });
            Notify("Saved " & Text(CountRows(changed)) & " change(s).", NotificationType.Success)
        )
    )
)

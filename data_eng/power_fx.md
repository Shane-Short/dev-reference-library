concat(
  'Module update applied: ',
  coalesce(body('GET_Assignment')?['Module_Name'], body('GET_Assignment')?['Module_ID'], 'Unknown Module'),
  ' | Adds: ', string(outputs('CMP_AddCount')),
  ' | Removes: ', string(outputs('CMP_RemoveCount')),
  ' | Updates: ', string(outputs('CMP_UpdateCount'))
)



<p><strong>Module Update Completed</strong></p>
<p><strong>Module:</strong> @{coalesce(body('GET_Assignment')?['Module_Name'], body('GET_Assignment')?['Module_ID'], 'Unknown')}</p>

<ul>
  <li><strong>Added</strong>: @{outputs('CMP_AddCount')}</li>
  <li><strong>Removed</strong>: @{outputs('CMP_RemoveCount')}</li>
  <li><strong>Skill Type Updates</strong>: @{outputs('CMP_UpdateCount')}</li>
</ul>

<p><strong>Assignment:</strong> @{body('GET_Assignment')?['ID']}</p>
<p><em>Timestamp:</em> @{utcNow()}</p>

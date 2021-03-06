Usage
*****

Simply add this recipe as a part in your buildout, specifying the target 
confguration file to be modified and the information to add. The marker 
lines can be customised to accommodate different commenting requirements. 
By default, a backup of the original configuation file is created before 
any changes are made, 

Supported options
=================

``target``

    Path to the file to be merged (required). If the file does not exist, it
    will be created unless create is set to False.
        
``section`` 
    
    A block of configuration text to place between the markers. You must 
    specify this or "section-file," or "section-template."
            
``section-file`` 

    A file to read, whose contents will be placed between the markers. This 
    is useful for more complex configurations. The file will be deleted after 
    use, unless delete-file is set to False. You must specify this or 
    "section," or "section-template."

``section-template`` 

    Set to the name of a section containing definitions for a 
    collective.recipe.template template. The template section need not be 
    added to the list of parts, unless you also want to execute it 
    separately. When invoked, the output file will be overridden and no file 
    will be created. You must specify this or "section," or "section-file."
    
    There are a few things to watch out for when using a template:
    
    - If you are not going to run the part, leave out the ``recipe`` 
      definition, or buildout will throw an error;
    - If you aren't going to run the part, you needn't specify an output file
      either. The recipe will supply a dummy one but no output file will be 
      created. If the part will be run, the output file you specify will not 
      be altered;
    - The section defining the ``configmanager`` options, **not** the 
      ``template`` options, will be the *base* section for the template. 
      That is, defining ``foo`` in the template section and using ``${foo}``
      in the template will fail. Simply use fully qualified placeholders 
      (``${section:foo}``) in your templates and everything will work 
      properly.

``allow-empty-section``

    Allow a section, section file or the results of a section-template to be 
    empty (after stripping leading and trailing whitespace). If this is the 
    case, the file will be left unchanged (but uninstall will still run on
    update to remove any existing entries).

    **Default:** False
    
``backup`` 
    
    Install and Update will create a backup which is the complete file name
    plus the extension ``.BK0``. Uninstall will create a backup with the 
    extension ``.BK1``. We do it this way because of the way buildout calls
    the install and uninstall routines (and the disconnect between them). 
    This approach ensures that we always have a valid backup, and that the 
    install backup won't overwrite a freshly created uninstall backup. 
    
    **Default:** True
    
``create``
    
    If the target file does not exist, create it and add the defined section 
    as the only contents. 
    
    **Default:** True

``uninstall`` 
    
    Remove section from file if part is uninstalled. If the file would be 
    empty after this it is deleted. If backup was originally set to True, 
    A backup will be created (see the note under ``start-marker`` for 
    caveats).
    
    **Default:** True
    
``insert-after``
    
    A regex pattern to look for in the target file. If found, the section 
    contents and markers will be inserted directly after this line. The regex 
    uses search, not match, so it will match the pattern anywhere in the 
    line. If whitespace or the location of the pattern in the line is 
    important, structure your regex accordingly. If the pattern is not found,
    the section will be appended to the end of the file, as usual.

    **Default:** None   
    
``replace``
    
    A regex pattern to look for in the target file. If found, this line will 
    be replaced by the section contents and markers. The regex 
    uses search, not match, so it will match the pattern anywhere in the 
    line. If whitespace or the location of the pattern in the line is 
    important, structure your regex accordingly. 
    
    If the pattern is not found, and ``insert-after`` was defined, that will 
    be will will be used to look for an insertion point. If ``insert-after`` 
    fails or was not defined, the section will be appended to the end of the 
    file, as usual.
    
    **Default:** None   

``start-marker``
    
    A line that marks the beginning of an auto-generated section. It should
    include a unique name (e.g. the section name) in case multiple sections 
    are added to the same file. You can do this by referencing 
    ``${:_buildout_section_name_}`` in your custom marker definition

    **Default:** # BEGIN - Generated by: ${:_buildout_section_name_}    

    .. note:: If you specify ``uninstall = false`` and later change the start-marker, the file won't be updated properly, as we rely on the uninstall routine to remove the previous markers.
    
``end-marker`` 

    The text to use for the ending marker line. 
    
    **Default:** # END - Generated by: ${:_buildout_section_name_}
        
``comment`` 

    A line that will be added directly after the start marker. If blank, it 
    will be omitted. 
    
    **Default:** # DO NOT EDIT: Text between these lines generated 
    automatically by buildout
            
``delete-file``
    If true, delete the file specified in ``section-file`` after processing. 
    
    **Default:** False
    
``strict``
    If true, treat all warnings, such as finding a start marker without a 
    matching end marker as errors.
    
    **Default:** False
    

Example usage
=============
We'll start with an existing file and add a section to it.

    >>> import os
    >>> from shutil import copy
    >>> test_path = join(os.path.dirname(__file__), 'testdata')
    >>> target_file = join(test_path, 'TEST_FILE.INI')
    >>> copy (
    ...     join(test_path, 'MASTER_TEST_FILE.INI'),
    ...     target_file
    ...     )
    
First we'll check that our data isn't in the file:

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# BEGIN' in contents
    False
    
And write out our configuration:

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... section =
    ...     four = 4
    ...     five = 5
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Installing...

We should now have a start marker, a comment, our new entries and our existing
ones:

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# BEGIN' in contents
    True
    >>> 'Text between' in contents
    True
    >>> 'four = 4' in contents
    True
    >>> '# END' in contents
    True
    >>> 'one = 1' in contents
    True
    >>> 'two = 2' in contents
    True

We always create a backup before doing anything, unless you explicity set
``backup = false.`` See the backup option documentation for details:

    >>> backup_file = join(test_path, 'TEST_FILE.INI.BK0')    
    >>> backup = open(backup_file, 'r')
    >>> contents = backup.read()
    >>> backup.close()

Our backup has only our original contents:    

    >>> 'one' in contents
    True
    >>> 'four' in contents
    False

Empty sections aren't allowed:

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... section =
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    While:
        ...

    Error:...

If this happens an already modified file won't be changed:

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# BEGIN' in contents
    True
    >>> 'four' in contents
    True
    
Unless we explicitly say so:

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... allow-empty-section = true
    ... target = %s
    ... section =
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

And now we see we have the markers but no data:    
    
    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# BEGIN' in contents
    True
    >>> 'one = 1' in contents
    True
    >>> 'four = 4' in contents
    False
    
Now let's change our section contents slightly and update our file

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... section =
    ...     four = 4
    ...     six = 6
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

We should now have everything we had before, but with 'five = 5' replaced by
'six = 6':

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# BEGIN' in contents
    True
    >>> 'Text between' in contents
    True
    >>> 'four = 4' in contents
    True
    >>> 'five' in contents
    False
    >>> 'six = 6' in contents
    True
    >>> '# END' in contents
    True
    >>> 'one = 1' in contents
    True
    >>> 'two = 2' in contents
    True
    
We can also look for a specific point to insert our contents:    

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... comment = 
    ... insert-after = two.*=.*
    ... section =
    ...     seven = 7
    ...     eight = 8
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

And we see that our section comes after ``two = 2`` and before `three = 3``:
    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> two_index = contents.find('two =')
    >>> three_index = contents.find('three =')
    >>> seven_index = contents.find('seven =')
    >>> two_index < three_index 
    True
    >>> two_index < seven_index 
    True
    >>> three_index > seven_index
    True        
    
And we can use insert-after if replace doesn't find anything:    
    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... comment = 
    ... replace = zero.*=.*
    ... insert-after = two.*=.*
    ... section =
    ...     nine = 9
    ...     ten = 10
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...    

And our section still comes after ``two = 2`` and before `three = 3``:
    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> two_index = contents.find('two =')
    >>> three_index = contents.find('three =')
    >>> nine_index = contents.find('nine =')
    >>> two_index < three_index 
    True
    >>> two_index < nine_index 
    True
    >>> three_index > nine_index
    True        
    
We can replace an existing line in a file:

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... comment = 
    ... replace = two.*=.*
    ... section =
    ...     eleven = 11
    ...     twelve = 12
    ... """ % (target_file)
    ... )
    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> original_two_index = contents.find('two =')
    >>> print system(buildout)
    Uninstalling...
    
And now our section replaces ``two = 2``
    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> two_index = contents.find('two =')
    >>> three_index = contents.find('three =')
    >>> begin_index = contents.find('# BEGIN')
    >>> eleven_index = contents.find('eleven =')
    >>> two_index == -1
    True
    >>> begin_index == original_two_index
    True
    >>> eleven_index < three_index 
    True    
      
We can also supply custom section markers:

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = /*START: ${:_buildout_section_name_}*/
    ... end-marker = /*FINISH: ${:_buildout_section_name_}*/
    ... comment = 
    ... section =
    ...     thirteen = 13
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

And we see our markers have changed

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# BEGIN' in contents
    False
    >>> '# END' in contents
    False
    >>> '/*START: config*/' in contents
    True
    >>> '/*FINISH: config*/' in contents
    True
    
Our section contents can come from a file:

    >>> section_file = join(test_path, 'SECTION_FILE.TXT')
    >>> copy (
    ...     join(test_path, 'MASTER_SECTION_FILE.TXT'),
    ...     section_file
    ...     )
    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = /*BEGIN: ${:_buildout_section_name_}*/
    ... end-marker = /*FINISH: ${:_buildout_section_name_}*/
    ... comment = 
    ... section-file = %s
    ... """ % (target_file, section_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

And our file now contains the settings from the input file:
    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# six' in contents
    False
    >>> 'fourteen = 14' in contents
    True
    >>> '// This' in contents
    True

Our input file can be deleted after use if we wish:

    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = /*BEGIN: ${:_buildout_section_name_}*/
    ... end-marker = /*FINISH: ${:_buildout_section_name_}*/
    ... comment = 
    ... section-file = %s
    ... delete-file = true
    ... """ % (target_file, section_file)
    ... )
    >>> print system(buildout)
    Uninstalling...    
    >>> os.stat(section_file)
    Traceback (most recent call last):
        ...
    OSError: ...

We can also use a template generated by collective.recipe.template. Note 
that we don't add the ``template`` section to the parts, as we aren't 
installing it on its own. If you did also want to generate an output file 
with the ``template`` part, you could certainly do so. You also don't need
to specify ``output`` or ``recipe``; we do here simply to show that no 
output file will be created:
    
    >>> output_file = join(test_path, 'OUTPUT.TXT')
    >>> template_file = join(test_path, 'TEMPLATE.IN')
    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [template]
    ... input = %s
    ... output = %s
    ... sixteen-var = 16
    ... seventeen-var = 17
    ...    
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = /*BEGIN: ${:_buildout_section_name_}*/
    ... end-marker = /*FINISH: ${:_buildout_section_name_}*/
    ... comment = 
    ... section-template = template
    ... """ % (template_file, output_file, target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...    

And our file now contains the settings from the template with the variables
inserted:

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '# seven' in contents
    False
    >>> 'sixteen = 16' in contents
    True
    >>> '// Test template' in contents
    True

And a template output file was not created:
    
    >>> os.stat(output_file)
    Traceback (most recent call last):
        ...
    OSError: ...

But if we want, we can run the template part as well, and a file will be
created:
    
    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config template
    ...
    ... [template]
    ... recipe = collective.recipe.template
    ... input = %s
    ... output = %s
    ... sixteen-var = 16
    ... seventeen-var = 17
    ...    
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = /*BEGIN: ${:_buildout_section_name_}*/
    ... end-marker = /*FINISH: ${:_buildout_section_name_}*/
    ... comment = // This is new
    ... section-template = template
    ... """ % (template_file, output_file, target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...    

The output file exists:

    >>> test = os.stat(output_file)
    
And our section was updated:    

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> '// This is new' in contents
    True
    
We don't have to have an input file to read from. If the file doesn't exist, 
we will create it (unless you specify ``create = false``):
    
    >>> os.remove(target_file)
    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = config
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = /*BEGIN: ${:_buildout_section_name_}*/
    ... end-marker = /*FINISH: ${:_buildout_section_name_}*/
    ... comment = 
    ... section =
    ...     eighteen = 18
    ...     nineteen = 19
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

Our file has our data, but nothing else:

    >>> target = open(target_file, 'r')
    >>> contents = target.read()
    >>> target.close()
    >>> 'one' in contents
    False
    >>> 'eighteen' in contents
    True
    
Finally, our section will be removed if the part is uninstalled. If the
result would be an empty file, the file will be removed. Just for fun we'll
change the marker definitions, to show that uninstall will still work:

    >>> os.remove(target_file)
    >>> write(
    ... 'buildout.cfg',
    ... """
    ... [buildout]
    ... newest = false
    ... parts = 
    ...
    ... [config]
    ... recipe = whtc.recipe.configmanager
    ... target = %s
    ... start-marker = //BEGIN: ${:_buildout_section_name_}
    ... end-marker = //FINISH: ${:_buildout_section_name_}
    ... comment = 
    ... section =
    ...     ten = 10
    ...     eleven = 11
    ... """ % (target_file)
    ... )
    >>> print system(buildout)
    Uninstalling...

The file is gone:

    >>> os.stat(target_file)
    Traceback (most recent call last):
        ...
    OSError: ...

But a backup (.BK1) was created, because ``backup = true`` by default:
    >>> backup_file = join(test_path, 'TEST_FILE.INI.BK1') 
    >>> backup = open(backup_file, 'r')
    >>> contents = backup.read()
    >>> backup.close()
    >>> print contents
    # Test file...

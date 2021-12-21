## Day 21

Today we're diving into Yara rules, a method for detecting malware by pattern recognition. Here's TryHackMe's breakdown for a Yara template:

    rule rulename
    {
        meta:
            author = "tryhackme"
            description = "test rule"
            created = "11/12/2021 00:00"
        strings:
            $textstring = "text"
            $hexstring = {4D 5A}
        conditions:
            $textstring and $hexstring
    }

The "meta" section is for metadata -- info about the rule. The string section can contain text strings in quotation marks or hex strings in braces. Text strings look to match content in any legible text inside a file; hex strings match a series of bytes.  

The "conditions" section should be familiar to any programmer, where you can use booleans to match a certain state. Above, "$textstring and $hexstring" means the rule will look to match both variables; if it was "$textstring or $hexstring" it would only look to match either one. "$texstring not $hexstring" would match only if the file contained the $textstring value but couldn't find the $hexstring value.  

NOTE: Make sure to check if the 0 you're typing into your string to match is actually an O.  

We'll write our own rule to match yesterday's EICAR anti virus test file:

    rule eicaryara {
        meta:
            author="tryhackme"
            description="eicar string"
        strings:
            $a="X5O"
            $b="EICAR"
            $c="ANTIVIRUS"
            $d="TEST"
        condition:
            $a and $b and $c and $d
    }

And save it as "eicaryara".  

Now let's take on today's challenges!  

>  We changed the text in the string $a as shown in the eicaryara rule we wrote, from X5O to X50, that is, we replaced the letter O with the number 0. The condition for the Yara rule is $a and $b and $c and $d. If we are to only make a change to the first boolean operator in this condition, what boolean operator shall we replace the 'and' with, in order for the rule to still hit the file? 

Well in that case, $a won't be found in the testfile. If we run `yara -s eicaryara testfile` we get this output:

    0x0:$a: X5O
    0x1c:$b: EICAR
    0x2b:$c: ANTIVIRUS
    0X35:$D: TEST

This shows us which strings matched and at which byte. The file starts with "X5O" so naturally it's found at position 0x0. But if we're looking for "X50" (with a zero) instead, that won't match the string. We need to change the condition to:

    $a or $b and $c and $d

That way, finding $a is optional.  

> What option is used in the Yara command in order to list down the metadata of the rules that are a hit to a file? 

That would be -m. If we run `yara -m eicaryara testfile` we get back:

    eicaryara [author="tryhackme",description="eicar string"] testfile

> What option is used in the Yara command in order to list down the metadata of the rules that are a hit to a file? 

This is the metadata section denoted by the heading "meta".  

> What option is used to print only rules that did not hit?

Whenever you're learning a new tool, it's always good to check it's `help` or `man` pages. We can run `yara --help` and see a brief rundown of useful options to use with Yara, including the one we're looking for:

    -n, --negate    print only not satisfied rules (negate)

> Change the Yara rule value for the $a string to X50. Rerun the command, but this time with the -c option. What is the result?

So we run `yara -c eicaryara testfile` and all we get back is:

    0

Huh? Check that help menu again: 

    -c, --count print only number of matches

Because our condition "$a and $b and $c and $d" evaluated to false, because "X50" is not in the file, we got no matches for our rule. If we changed the condition to the one we thought of earlier, "$a or $b and $c and $d", then ran `yara -c eicaryara testfile`, we'd get back:

    1

Because it matched one of our rules.  

If any of this blue team stuff gets you going, there's a free room on TryHackMe for more Yara stuff [here](https://tryhackme.com/room/yara).  
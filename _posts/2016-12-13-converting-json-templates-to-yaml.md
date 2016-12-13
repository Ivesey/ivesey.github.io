---
layout: post
title:  Converting CloudFormation JSON templates into YAML
subtitle: And making them look good
date:   2016-12-13
category: aws
tags: [cloudformation, yaml]
---

[Originally posted on qa.com](https://www.qa.com/blogs/converting-cloudformation-json-templates-into-yaml 'Original post'). Reposted here because frankly, it's easier for me to update it and Jekyll makes code snippets look good.

Ever since YAML support in CloudFormation (hereafter referred to as 'CFn') templates was announced, I've developed a new hobby: converting templates from JSON to YAML.

Delegates keep asking me why I'm so excited about YAML in CFn and the simple answer is that it's easier for humans to read, and you can put comments in more easily. But the more complex answer is that it's also easier for humans to write and that user-data is so much clearer now.

However, I keep running into little road bumps in my conversions, so this post is partly for my benefit as a reminder of what I've discovered.

For this post, I'll use the example of converting [the WordPress single instance sample template](https://s3-eu-west-1.amazonaws.com/cloudformation-templates-eu-west-1/WordPress_Single_Instance.template 'WordPress sample template'). this is the one in eu-west-1, but there's a copy in every region.

At the end, I was going to compare the number of lines of code ('LOC') between the original JSON and the new-and-improved YAML, but then I remembered that there's a stupidly long set of mappings in this template, including a completely unused one for NAT instances but also the Linux AMI mappings (and AllowedValues) cover every single instance type. Are you seriously going to run WordPress on a cc2.8xl?!?!?! So I might cheat and compare LOC between everything except the Mappings section and the AllowedValues for InstanceType...

NOTE: I've got that re:Invent feeling - that as I'm writing this, someone from AWS is preparing to announce a tool that does awesome and intelligent conversions from JSON-Cfn to YAML-Cfn. EDIT: Nope, I'm good!

## Step 1: Convert the JSON to YAML
Clearly, as YAML is a super-set of JSON, there are many ways to perform the initial conversion, from online tools to Python libraries, but I use [Atom](http://atom.io 'Atom homepage') a lot these days and there's a handy add-in for that called json-converter ('j-c'). You can install it by going to File | Settings | Install or at the command line with `apm install json-converter`

## Step 2: Remove unnecessary quotes
We'll treat quotes around intrinsic functions as necessary for now, because we also want to clean those up in our subsequent passes.

_Some_ quotes are clearly necessary. Working out which ones are which is not straightforward, but it's all about which characters have special meaning in YAML. There's two ways to do this; you can either remove them all using find and replace and then discover which ones were necessary by validating the template repeatedly: `aws cloudformation validate-template --template-body file://path/to/template.yaml`

Or, you can base it on this (utterly incomplete) list of necessary ones:

- ones around a RegEx containing square brackets `[]` (i.e. SSHLocation.AllowedPattern doesn't need them, DBName.AllowedPattern does)
- single asterisks `*` in IAM policies etc. (no example in the WP template)

Here's a list of ones that are OK:

- CFn types (i.e. AWS::EC2::KeyPair::KeyName) - j-c leaves quotes because they contain `:`s
- dates (i.e. AWSTemplateFormatVersion)
- whole numbers (i.e. DBName.MinLength)
- booleans (i.e. DBUser.NoEcho)
- urls (i.e. http://wordpress.org/latest.tar.gz)

And ones I suspect are required so haven't bothered to test:

- masks (i.e. mode in file in cfn-init) - I suspect leading zeros would be ignored if YAML thought it was a number not a string.

## Step 3: Tidy up those `Fn::FindInMap`s for ImageIds
We have some nasty nested ones in the WordPress template, but WebServer | Properties | ImageId in JSON looks like this:

{% highlight json %}
"ImageId" : { "Fn::FindInMap" : [ "AWSRegionArch2AMI", { "Ref" : "AWS::Region" },
                  { "Fn::FindInMap" : [ "AWSInstanceType2Arch", { "Ref" : "InstanceType" }, "Arch" ] } ] },
{% endhighlight %}

And we can make it look much nicer:
{% highlight yaml %}
ImageId: !FindInMap
  - AWSRegionArch2AMI
  - !Ref AWS::Region
  - !FindInMap
      - AWSInstanceType2Arch
      - !Ref InstanceType
      - Arch
{% endhighlight %}

Or, possibly:
{% highlight yaml %}
ImageId: !FindInMap
  - AWSRegionArch2AMI
  - !Ref AWS::Region
  - !FindInMap [AWSInstanceType2Arch, !Ref InstanceType, Arch]
{% endhighlight %}

Or, let's really go for LOC reduction now! Note the quotes. We're getting very nest-y now and I think CFn / YAML is starting to struggle:
{% highlight yaml %}
ImageId: !FindInMap [AWSRegionArch2AMI, !Ref "AWS::Region", !FindInMap [AWSInstanceType2Arch, !Ref InstanceType, Arch]]
{% endhighlight %}

## Step 4: Tidy up those function calls in user-datas and cfn-inits
First, a note about string literals in YAML.

The pipe character `|` tells YAML to preserve newlines, which means we can lose a lot of `\n`s.

The `>` character tells it to remove newlines, which means we can wrap stuff.

The `-` tells YAML to lose trailing newlines at the end of the string.

The `+` tells YAML to preserve trailing newlines. Very useful in some instances.

Generally speaking, we can eliminate all those `Fn::`s and replace them with `!`s, but there are restrictions in CFn around nesting short-form function calls, presumably to do with how the template is being pre-processed by CFn, so again it's worth validating your templates after every tweak. Leave `Fn::Base64`s alone as a rule.
But for the simplest examples, we can replace `Fn::Join`s and replace them with `!Sub`s, which makes most files and outputs so much neater. Also, using `!Sub`s, we can almost entirely eliminate `Fn::GetAtt`s.

### Simple Example 1: Losing joins and refs

#### WebServer | Metadata | AWS::CloudFormation::Init | install_cfn | files | cfn-hup.conf | content
In the original JSON (clearly the original template author has tried to make it as legible as possible with the formatting):
{% highlight json %}
"content": { "Fn::Join": [ "", [
  "[main]\n",
  "stack=", { "Ref": "AWS::StackId" }, "\n",
  "region=", { "Ref": "AWS::Region" }, "\n"
]]},
{% endhighlight %}
J-c is seeing those newlines and trying to preserve that formatting, which looks a bit messy, but we can lose the join completely, so:
{% highlight yaml %}
content:
  'Fn::Join':
    - ''
    - - |
        [main]
      - stack=
      - Ref: 'AWS::StackId'
      - |+

      - region=
      - Ref: 'AWS::Region'
      - |+
{% endhighlight %}

becomes the much neater:

{% highlight yaml %}
content: !Sub |
  [main]
  stack=${AWS::StackId}
  region=${AWS::Region}
{% endhighlight %}

Note the `|` to preserve the newlines

### Simple Example 2: Losing `Fn::GetAtt`s

#### Outputs | WebsiteURL | Value
The original JSON:
{% highlight json %}
"Value" : { "Fn::Join" : ["", ["http://", { "Fn::GetAtt" : [ "WebServer", "PublicDnsName" ]}, "/wordpress" ]]},
{% endhighlight %}

Isn't visually enhanced by j-c. Replace the `Fn::GetAtt` and the `Fn::Join` so that:
{% highlight yaml %}
Value:
  'Fn::Join':
    - ''
    - - 'http://'
      - 'Fn::GetAtt':
          - WebServer
          - PublicDnsName
      - /wordpress
{% endhighlight %}

becomes the much, _much_ neater:
{% highlight yaml %}
Value: !Sub
  http://${WebServer.PublicDnsName}/wordpress
{% endhighlight %}

### Trickier Example 3: Interesting formatting

#### WebServer | Metadata | AWS::CloudFormation::Init | install_cfn | files | /etc/cfn/hooks.d/cfn-auto-reloader.conf | content
Two tidy-up options here, depending on whether you care about line wrapping. Here's the original JSON version:
{% highlight json %}
"content": { "Fn::Join": [ "", [
  "[cfn-auto-reloader-hook]\n",
  "triggers=post.update\n",
  "path=Resources.WebServer.Metadata.AWS::CloudFormation::Init\n",
  "action=/opt/aws/bin/cfn-init -v ",
          "         --stack ", { "Ref" : "AWS::StackName" },
          "         --resource WebServer ",
          "         --configsets wordpress_install ",
          "         --region ", { "Ref" : "AWS::Region" }, "\n"
]]},
{% endhighlight %}

Here's the j-c converted version (j-c is trying to preserve the formatting as much as possible with the pipes and the angle brackets and the quotes):
{% highlight yaml %}
content:
  'Fn::Join':
    - ''
    - - |
        [cfn-auto-reloader-hook]
      - |
        triggers=post.update
      - >
        path=Resources.WebServer.Metadata.AWS::CloudFormation::Init
      - 'action=/opt/aws/bin/cfn-init -v '
      - '         --stack '
      - Ref: 'AWS::StackName'
      - '         --resource WebServer '
      - '         --configsets wordpress_install '
      - '         --region '
      - Ref: 'AWS::Region'
      - |+
{% endhighlight %}

And here's your conversion if you're not worried about line wrapping [which will ironically be wrapped when you see this post]:
{% highlight yaml %}
content: !Sub |+
  [cfn-auto-reloader-hook]
  triggers=post.update
  path=Resources.WebServer.Metadata.AWS::CloudFormation::Init
  action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServer --configsets wordpress_install --region ${AWS::Region}
{% endhighlight %}

And if you do care, and don't mind putting `\`s in commands, it gets even better!
{% highlight yaml %}
content:
  !Sub |+
    [cfn-auto-reloader-hook]
    triggers=post.update
    path=Resources.WebServer.Metadata.AWS::CloudFormation::Init
    action=/opt/aws/bin/cfn-init -v \
             --stack ${AWS::StackName} \
             --resource WebServer \
             --configsets wordpress_install \
             --region ${AWS::Region}
{% endhighlight %}

### Trickier Example 4: escaping single-quote literals

#### WebServer | Metadata | AWS::CloudFormation::Init | install_wordpress | files | /tmp/setup.mysql
Decipherable JSON:
{% highlight json %}
"content" : { "Fn::Join" : ["", [
  "CREATE DATABASE ", { "Ref" : "DBName" }, ";\n",
  "CREATE USER '", { "Ref" : "DBUser" }, "'@'localhost' IDENTIFIED BY '", { "Ref" : "DBPassword" }, "';\n",
  "GRANT ALL ON ", { "Ref" : "DBName" }, ".* TO '", { "Ref" : "DBUser" }, "'@'localhost';\n",
  "FLUSH PRIVILEGES;\n"
{% endhighlight %}
So those single-quotes cause problems for us and we'll need to escape them. Or will we? I need to test my template out I guess...
The following validates OK, but I have a feeling it won't build because of those `@`s. OMG, yes it does! Perhaps I should re-classify this as not particularly tricky!
{% highlight yaml %}
content: !Sub |
  CREATE DATABASE ${DBName};
  CREATE USER '${DBUser}'@'localhost' IDENTIFIED BY '${DBPassword}';
  GRANT ALL ON ${DBName}.* TO '${DBUser}'@'localhost';
  FLUSH PRIVILEGES;
{% endhighlight %}

### Trickier Example 5: Multiple lines of "interesting" formatting

#### WebServer | Properties | UserData
JSON:
{% highlight json %}
"Fn::Base64" : { "Fn::Join" : ["", [
               "#!/bin/bash -xe\n",
               "yum update -y aws-cfn-bootstrap\n",

               "/opt/aws/bin/cfn-init -v ",
               "         --stack ", { "Ref" : "AWS::StackName" },
               "         --resource WebServer ",
               "         --configsets wordpress_install ",
               "         --region ", { "Ref" : "AWS::Region" }, "\n",

               "/opt/aws/bin/cfn-signal -e $? ",
               "         --stack ", { "Ref" : "AWS::StackName" },
               "         --resource WebServer ",
               "         --region ", { "Ref" : "AWS::Region" }, "\n"
]]}
{% endhighlight %}

Again, depends on your formatting preferences. If you're avoiding long lines, you have a couple of options; one involves more `!Join`ing than the other. I've preferred the version that uses less, on the basis that if you're basically copying-and-pasting this stuff and you care about text wrapping, you'll already be using `\`s. So here's that user data:
{% highlight yaml %}
Fn::Base64: !Sub |+
  #!/bin/bash -xe
  yum update -y aws-cfn-bootstrap
  /opt/aws/bin/cfn-init -v \
      --stack ${AWS::StackName} \
      --resource WebServer \
      --configsets wordpress_install \
      --region ${AWS::Region}
  /opt/aws/bin/cfn-signal -e $? \
      --stack ${AWS::StackName} \
      --resource WebServer \
      --region ${AWS::Region}
{% endhighlight %}

Without the `\`s I'd need more joins because the leading `-`s would cause problems. My final minified version will have a "don't care about wrapping" version.

## Step 5: Tidy up all the remaining `Ref:`s
This could arguably be done at step 2.5 or later, but I've kept it until last on the grounds that if I'd said to do this earlier, and you were following along blindly and find-and-replacing, it might have made a mess of our user-datas and cfn-inits.

So `!Ref`, as opposed to `Ref:`, doesn't need to be a separate key, meaning we can save lots of lines. What I mean is WebServerSecurityGroup &#124; Properties &#124; SecurityGroupIngress &#124; CidrIp can go from this:
{% highlight yaml %}
CidrIp:
  Ref: SSHLocation
{% endhighlight %}

to this:
{% highlight yaml %}
CidrIp: !Ref SSHLocation
{% endhighlight %}

There aren't many in the WordPress template, and make sure you don't accidentally remove a newline / hyphen from one that's part of an array (such as WebServer &#124; Properties &#124; SecurityGroups in this case), but that could save a lot of LOC in some templates.

## And the final line count

|---
| Format | LOC | Ignored<sup>1</sup> | Adjusted |
|-|-:|-:|-:
| JSON | 361 | 1+129 = 130 | 231 |
| YAML | 526 | 59+261+1+6+4 = 331 | 195 |
| YAML-min | 500 | 59+261 = 320 | 180 |
{:.mbtablestyle}

<sup>1</sup>Mappings, AllowedValues and my comments in the non-minified version are ignored

## Finally, a couple of other tricks, including FindInMap in Sub
Something I've been trying to get to work for a while, and seen floating around the interwebs, is including a `Fn::FindInMap` in a `Fn::Sub` (or indeed a `!FindInMap` in a `!Sub`) in a CloudFormation template. I wanted to use it to create a shell script which invokes run-instances as part of a cfn-init section, but it could also prove useful in a user-data section. It's not relevant to the WordPress template, but it's been bugging me for a while. So I solved it today, which is what prompted this blog in the first place.
I banged my head against several brick walls while working on this, but I believe I have a general format now.
Replacing `Fn::GetAtt`s with placeholders is one thing, but replacing them with what [the AWS documentation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html 'AWS Documentation on Sub') confusingly refers to as "mappings" [not to be confused with ['Mappings'](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/mappings-section-structure.html 'AWS Documentation on Mappings')] is another level. I'll present it without the rabbit-holes I fell into.
So according to the afore-mentioned documentation, one can substitute a `${pseudo-parameter}`, a `${resource.attribute}`, a `${mapping}`, a `${!literal-string-do-not-substitute-this}` in a `!Sub`. Sadly, there's no documentation about how to do that with a FindInMap, hence this post. I couldn't work out how to do it inline, despite trying multiple combinations of colons and quotes, so I decided that I'd need to use a mapping. Now, such an item will have one of two forms; either a one line sub or a multiline sub. Chances are you're looking for a mutliline solution.
NOTE: I haven't actually tried this with user-data yet, only on `Output`s, but the principal should be globally applicable. Just don't blame me when the wheels fall off.

### Single-liner
The `!Sub` function wants two arguments (an array) passed to it, but `Fn::FindInMap` has some odd behaviour alluded to above. Arg0 contains the mapping, arg1 the map key and its value. This is where things normally go wrong.

{% highlight yaml %}
FindInMapInSub1Line:
  Value: !Sub
    - I've been subbed ${Subbed}
    - Subbed:
        Fn::FindInMap: [RegionOS2AMI, Ref: "AWS::Region", Windows]
{% endhighlight %}

### Multi-liner
Buoyed by my success with a single line, I went over various iterations trying to get it to work with multiline strings, failing spectacularly until I realised that the validation errors I was getting were actually telling me what the problem was. Again it's down to the order in which things are evaluated, so here's a successful multiliner:

{% highlight yaml %}
FindInMapInSubMultiline:
  Value: !Sub
    - |
      This is line one
      And line ${Subbed2} two
    - Subbed2:
        Fn::FindInMap:
          - RegionOS2AMI
          - !Ref AWS::Region
          - Windows
{% endhighlight %}

Note the pipe's location at the start of arg0, rather than where I had it originally which was after the `!Sub`. Note also the multiline `Fn::FindInMap`; the single line version from the previous example should work equally well.

#### But do you really need it?
In this case, no I don't. AMI Mappings are so last decade. The modern approach would be to use a `CustomResource` to look up the AMI. [Here's a walkthrough of creating such a thing](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/walkthrough-custom-resources-lambda-lookup-amiids.html 'AMI Lookup lambda Walkthrough'). And then my `!Sub` would be as simple as:
{% highlight yaml %}
aws ec2 run-instances --image-id ${AMILookup.Id}
{% endhighlight %}

I hope you find this useful.

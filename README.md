# volsink
volsink is a tool that leverages volatility3 to perform analysis on Windows memory images. During an investigation there is a lot of time pressure to figure out what happened and report up to the management team.
Volatility3 was chosen as it automatically detects the version of Operating System in use. Volatility2 has not been actively maintained for many years, and quite a few plugins are no longer supported by the third party developers who wrote them.

Volsink helps you get answers more quickly by using a wizard-driven menu (still a work in progress), then runs volatility plugins and dumps the output to individual text files for analysis.

There are 2 modes: basic (quick) and advanced (slower). Basic runs the pstree,pslist, (and a few others) then dumps the output to an individual text file.

Advanced throws the lot in there, hence the name volsink. I'm all about efficiency, so calling the tool *volkitandkaboodle* didn't seem helpful to my fellow analysts and incident responders.

This also dumps the output to a text file, which each plugin forming the filename.

Use volsink as a quick first pass for your analysis. After reviewing the output, use volatility surgically to target modules,strings and offsets of interest.


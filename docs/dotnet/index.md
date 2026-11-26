# .NET Malware 101

> How .NET malware is compiled and run by the CLR, and how to navigate a .NET assembly’s metadata, tokens and method bodies with dnSpy, ILSpy and PEStudio.

- Source: https://research.intezer.com/docs/dotnet/
- Published: 2024-03-13
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


The .NET framework gives malware authors the same things it gives legitimate
developers: fast development, a rich class library, and tight Windows
integration. Ransomware, credential stealers, wipers, and remote access trojans
are all routinely written in .NET, which makes reading compiled .NET assemblies
a practical skill for anyone triaging Windows threats.

This section covers what a .NET binary actually contains and how to navigate it
with tools like dnSpy, ILSpy, and PEStudio.

- [Analyzing the .NET Executable File Structure](exe-structure/) - How managed
  code is compiled and executed by the CLR, what lives inside a .NET assembly,
  and a guided tour of the CLR runtime header, metadata streams, metadata
  tables, tokens, the manifest, and method body headers, using the Sunburst
  (SolarWinds) sample as the worked example.

## Pages in this section

- [Analyzing the .NET Executable File Structure](https://research.intezer.com/docs/dotnet/exe-structure/): Inside the .NET executable format: CLR compilation and runtime, metadata streams and tables, tokens, the manifest and method bodies, using the Sunburst sample.


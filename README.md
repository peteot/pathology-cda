CDA Rendering of Anatomic Pathology Synoptic Reports
====================================================

_Peter O'Toole, VP, Engineering at mTuitive, Inc._

[HL7 Clinical Document Architecture (CDA)](http://www.hl7.org/implement/standards/product_brief.cfm?product_id=7) has been around for many years, but recently it has resurfaced as part of meaningful use standards for exchanging clinical summaries.  In addition, the Consolidated CDA (C-CDA) [Implementation Guide](http://www.hl7.org/implement/standards/product_brief.cfm?product_id=258) from HL7 helps define best practices for structuring several clinical document types, like discharge summaries and operative reports. Recently we did some work with the [SMART C-CDA Collaborative](http://smartplatforms.org/2013/07/introducing-the-smart-c-cda-collaborative/) on operative reports, and it got us thinking about how we would provide a similar CDA implementation for synoptic pathology reports.

CDA is a very expressive language for describing highly structured clinical documents.  It is very flexible, which helps when taking the standard into new areas, but its flexibility also  means that no two organizations are likely to come up with the same way of structuring the same documents. Hence the C-CDA Implementation Guide, which provides further guidance on implementing common clinical documents using CDA.

In anatomic pathology, it is required that pathologists describe tumors using peer-reviewed, established protocols from the College of American Pathologists. The [CAP Cancer Checklists](http://www.cap.org/apps/cap.portal?_nfpb=true&cntvwrPtlt_actionOverride=/portlets/contentViewer/show&_windowLabel=cntvwrPtlt&cntvwrPtlt{actionForm.contentReference}=committees/cancer/cancer_protocols/protocols_index.html&_pageLabel=cntvwr) are also published in electronic form as the [CAP eCC](http://www.cap.org/apps/cap.portal?_nfpb=true&cntvwrPtlt_actionOverride=%2Fportlets%2FcontentViewer%2Fshow&_windowLabel=cntvwrPtlt&cntvwrPtlt%7BactionForm.contentReference%7D=snomed%2Fabout_ecc.html&_state=maximized&_pageLabel=cntvwr).  eCC has been instrumental in enabling organizations like NAACCR and Cancer Care Ontario to collect standardized, detailed pathologic cancer data at a large scale.  This type of structured pathology reporting is often called synoptic reporting.

Currently, most transmission of eCC uses an HL7 v2.5 format defined in NAACCR Volume V.  As an exercise, we at mTuitive thought it would be productive (and fun) to explore a rendering of eCC as CDA documents, following the same fundamental rules as NAACCR Volume V.  Although the C-CDA guide does not include anatomic pathology reports, the CDA standard is perfectly well suited to it.  Others have created implementation guides and templates for rendering AP reports as CDA documents, though the focus is usually on the entire traditional AP report, including narrative history and gross description. Our work here only includes representing the synoptic portion of an AP report, specifically CAP eCC documents.

##Implementation Details

[mTuitive xPert](http://www.mtuitive.com/cancer-reporting/) is a synoptic reporting application for pathologists that interoperates with traditional anatomic pathology (AP) systems. As such, the product always needs to communicate its results back to its host AP system.  Most often, this is achieved with a traditional HL7 v.2.x message.  NAACCR Volume V also uses HL7 v.2.5 because of its ubiquitous presence in the industry. Any CDA implementation must incorporate all of the same data included in the current NAACCR standard, including message headers and report data.

The message headers that include patient and order information are straightforward to implement with CDA.  The [IHE Anatomic Pathology Technical Framework](http://www.ihe.net/Technical_Framework/upload/IHE_PAT_Suppl_APSR_Rev1-1_TI_2011_03_31.pdf) already provides all the guidance necessary to implement the document header using CDA, including useful mappings from HL7 v2.5.  The header will ensure the document identifies the correct patient, specimen, pathologist, etc. The focus of our exploration is the structured data payload of a synoptic report. 

Internally, xPert produces a proprietary XML representation of a synoptic report.  The structured elements of the CDA report can be produced directly from this internal XML document.  For this exploration, an XSLT transformation is applied that can transform any xPert eCC report into a CDA document's structuredBody, the part of a CDA that contains the individual data elements.  This is encouraging because the ability to do perform this transformation within the constraints of XSLT reassures us that the mapping from eCC to CDA is straightforward, although a production implementation might use a more traditional programming language. XSLT also provides an easy way to generate sample documents for standards development work.

The [NAACCR Volume V](http://www.naaccr.org/LinkClick.aspx?fileticket=KA9zQRZ5Vn8%3d&tabid=136&mid=476) format represents every question and answer from an eCC with an OBX segment.  In this example, the question, "Specimen Laterality" is preceded by its CAP eCC CKEY (unique identifier).  The answer, "Right" is also preceded by its CKEY.  In both cases, the third component identifies the coding system, in this case "CAPECC", shorthand for CAP eCC.

    OBX|9|CWE|18251.1000043^Specimen Laterality^CAPECC||18254.1000043^Right^CAPECC

Following the work done here by NAACCR, we can map individual questions, answers, and codes onto CDA structures.  The basic unit of structured data in CDA is the Section.  Here is a simple rendering of the above in CDA:

    <component>
      <section>
        <code code="18251.100004300" codeSystem="2.16.840.1.113883.3.1434"
              codeSystemName="CAPECC" displayName="Specimen Laterality" />
        <title>Specimen Laterality</title>
        <text>
          <list>
            <item>Right</item>
          </list>
        </text>
        <entry>
          <observation classCode="OBS" moodCode="EVN">
            <code code="18254.100004300" codeSystem="2.16.840.1.113883.3.1434"
                  codeSystemName="CAPECC" displayName="Right" />
            <text>Right</text>
          </observation>
        </entry>
      </section>
    ... child sections ...
    </component>

Above, you will see each question, answer, and CKEY represented in XML. Every code system in CDA must be represented by an OID registered with HL7. For this we used CAP's OID, though it is not clear if that covers the CKEY coding scheme or if another OID would need to be registered.  In addition, CDA allows you to render the human-readable text in the <text> element, and the coded, structured data in the <entry> element.  This means we can make people and machines happy with a single structured xml document!

Physical quantities are another example:

    OBX|17|NM|1931.1000043^Maximum Tumor Thickness (Note D)^CAPECC|18819|2.1|mm^millimeter^UCUM

The above identifies the section the same way, but also includes a number value and units identified using the UCUM system.  Here is the same rendered in CDA:

    <section>
      <code code="18819.100004300" codeSystem="2.16.840.1.113883.3.1434"  
            codeSystemName="CAPECC" displayName="Maximum Tumour Thickness (mm)" />
      <title>Specify (mm)</title>
      <text>2.1</text>
      <entry>
        <observation classCode="OBS" moodCode="EVN">
          <code nullFlavor="UNK" />
          <text>2.1 mm</text>
          <value xsi:type="PQ" value="2.1" unit="mm" />
        </observation>
      </entry>
      .... child sections ....
    </section>

A full example document is attached.

##Conclusion

The result is that a CDA eCC document offers several advantages over the HL7 v2.5 model.  The CDA document contains all of the same information as the HL7 v2.5 rendering, but in an XML format that is more familiar to today's developers. In addition, the CDA can potentially handle more information if available, like representing a concept with more than one code (ICD-9 and ICD-10, for example).  Of course, HL7 v2.5 has the advantage of working with almost every system on the market, lowering the barrier to adoption.  A CDA rendering of eCC may be very useful for organizations who are already doing work with the CDA standard, or as a complement to the HL7 v2.5 messages for use with some systems that are interested in receiving CDA or XML.  (Indeed, rules for transmitting a CDA document within an HL7 v.2.x message do exist.) In short, we feel further work toward an implementation guide for synoptic pathology reports can lead to very useful results and also brings electronic pathology reporting more in line with current trends in the era of meaningful use. We welcome feedback and participation from the community toward this common goal.

##Open Questions

We look forward to input from other organizations and CDA experts. A couple of initial questions for the community:

1. Is the CAP's OID the appropriate code system identifier for eCC CKEYs?
2. Is the concatentation of the CAP's OID with an eCC template ID, an appropriate CDA template id for each tumor site?

##References
 
CAP electronic Cancer Checklists (CAP eCC)
http://www.cap.org/apps/docs/snomed/documents/about_cap_ecc.pdf

HL7 CDA Release 2
http://www.hl7.org/implement/standards/product_brief.cfm?product_id=7

HL7 Implementation Guide for CDA® Release 2: IHE Health Story Consolidation, Release 1.1 - US Realm
http://www.hl7.org/implement/standards/product_brief.cfm?product_id=258

IHE Anatomic Pathology Technical Framework Supplement, Anatomic Pathology Structured Reports
http://www.ihe.net/Technical_Framework/upload/IHE_PAT_Suppl_APSR_Rev1-1_TI_2011_03_31.pdf

NAACCR Volume V, Pathology Laboratory Electronic Reporting, Version 4.0
http://www.naaccr.org/LinkClick.aspx?fileticket=KA9zQRZ5Vn8%3d&tabid=136&mid=476

##Examples

Included is a full example of a melanoma CDA XML file.  Also included is an html representation of that report, rendered using the standard CDA xsl style sheet from HL7.




























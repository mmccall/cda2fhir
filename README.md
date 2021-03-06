<!--
Copyright (C) 2019 Amida Technology Solutions, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

cda2fhir 
===
[![License Info](http://img.shields.io/badge/license-Apache%202.0-brightgreen.svg)](https://github.com/srdc/cda2fhir/blob/master/LICENSE.txt)
[![Jenkins CI](https://jenkins.amida.com/buildStatus/icon?job=CDA2FHIR%20Tests/)](https://jenkins.amida.com/job/CDA2FHIR%20Tests/)

## Overview
cda2fhir is a Java library to transform HL7 [CDA R2](https://www.hl7.org/implement/standards/product_brief.cfm?product_id=7) instances to HL7 [FHIR](https://www.hl7.org/fhir/) resources. More specifically, cda2fhir enables automatic transformation of
Consolidated CDA (C-CDA) Release 2.1 compliant documents to corresponding FHIR STU3 resources. For this purpose, cda2fhir provides extensible document transformers, resource transformers, data type transformers and value set transformers.

The current implementation provides a
document transformer for the Continuity of Care Document (CCD) template, but further document transformers, e.g. for Discharge Summary or Referral Note,
can be easily introduced by reusing the already existing section and entry transformers. Although the cda2fhir library expects C-CDA R2.1 compliant documents/entries, it has been tested as well with several older document instances compliant with earlier releases of C-CDA. The [HAPI FHIR Validator](http://hapifhir.io/doc_validation.html) is also integrated for automated validation of the generated FHIR resources.

## Latest Updates
The original cda2fhir library created by [SRDC](https://github.com/srdc/cda2fhir) mapped C-CDA resources to FHIR DSTU2-compliant resources. [Amida](https://www.amida.com) has created this fork of this library, incorporating the work of [CarthageKing](https://github.com/CarthageKing/cda2fhir), to instead map C-CDA resources to FHIR STU3-compliant resources. [Model Driven Health Tools (MDHT)](https://projects.eclipse.org/projects/modeling.mdht) is used for CDA manipulation and
[HAPI](http://hapifhir.io/) is used for FHIR manipulation. This version of cda2fhir currently supports the following C-CDA Section to Resource mappings:

|C-CDA Section    | FHIR Resource(s)   |
|------------------|-----------------|
|Medications           |MedicationStatement, MedicationRequest, MedicationDispense, Medication|
|Procedures           |Procedure|
|Immunizations |Immunization, Medication|
|Results| DiagnosticReport, Observation|
|Vital Signs| Observation|
|Problems (Conditions)| Condition|
|Allergies and Intolerances| AllergyIntolerance|
|Encounters| Encounter|

In addition to the above mappings, cda2fhir also uses and supports the Patient, Practitioner, PractitionerRole, Organization, Location, Device, DocumentReference, Composition, and Provenance FHIR Resources. Field-level mappings for the aformentioned Resources may be found within [this spreadsheet](https://docs.google.com/spreadsheets/d/1fyoUSfTXBUJMC2wqqC1hAda77pyHWpULqrsF6G3Aock/edit?usp=sharing).

**This version of the cda2fhir library implements several additional features not present in the SRDC version. These include:**

* cda2fhir is now capable of generating "transactional" bundles.
* cda2fhir now supports the generation of Provenance objects, optionally taking in an Identifier resource and string representation of the source file to generate the accompanying Device and DocumentReference resources respectively.
* Bundles now de-duplicate against themselves for certain resources; this is done to prevent duplicate resources from being created in a FHIR server, and works together with the ifNoneExist parameters.
  * Medications are de-duplicated based on their encoding, and their manufacturing organization.
  * Organizations and Practitioners are de-duplicated based on identifier.
* Bundles now use the "ifNoneExist" parameter to prevent duplicate resources from being created on a target FHIR server. This parameter uses the identifier field to prevent duplicates for all resources, with the exception of:
  * Medications are de-duplicated based on their encoding.
  * Provenance and Composition resources, which are not de-duplicated for attribution purposes.
  * DocumentReference resources are not de-duplicated, as there is no query accessor for the attachment hash field.
* An integration test now uses Docker to automatically provision a HAPI FHIR server, post a transactional bundle to it, and.spot check for issues. Once complete this process will automatically de-provision the server.

## Installation

This project is built in Java, using version 1.8, and uses Apache Maven for dependency management. Please visit [Maven's website](http://maven.apache.org/) in order to install Maven on your system. To run the project's tests, your system will require Docker; please visit [Docker's website](https://www.docker.com/) for installation instructions.

Under the root directory of the cda2fhir project run the following:

	$ cda2fhir> mvn install

In order to make a clean install run the following:

	$ cda2fhir> mvn clean install

These will build the cda2fhir library and also run a number of test cases, which will transform some C-CDA Continuity of Care Document (CCD) instances,
and some manually crafted CDA artifacts (e.g. entry class instances) and datatype instances to corresponding FHIR resources.

This project incrementally builds and releases files for use in maven projects, using the instructions provided [here](./doc/maven-instructions.md). To use, add the repository and dependency to your pom.xml like so, replacing the `X.Y.Z` with a version number.

```
<repository>
  <id>amida-github</id>
  <name>github</name>
  <url>https://github.com/amida-tech/cda2fhir/raw/release</url>
</repository>
...
<dependency> 
  <artifactId>cda2fhir</artifactId>
  <groupId>tr.com.srdc</groupId>
  <version>X.Y.Z</version>	        
</dependency>
```

## Transforming a CDA document to a Bundle of FHIR resources

The below code is an annotated example of a basic CCD document transformation, further code examples can be found in [CCDTransformerTest.java](./src/test/java/tr/com/srdc/cda2fhir/CCDTransformerTest.java) file. You may also review all implemented interfaces in the [CCDTransformerImpl.java](./src/main/java/tr/com/srdc/cda2fhir/transform/CCDTransformerImpl.java) file.

The output of this operation will be located at: `src/test/resources/output/C-CDA_R2-1_CCD.json`.

```java
// Load MDHT CDA packages. Otherwise ContinuityOfCareDocument and similar documents will not be recognised.
// This has to be called before loading the document; otherwise will have no effect.
CDAUtil.loadPackages();

// Read a Continuity of Care Document (CCD) instance, which is the official sample CCD instance
// distributed with C-CDA 2.1 specs, with a few extensions for having a more complete document
FileInputStream fis = new FileInputStream("src/test/resources/C-CDA_R2-1_CCD.xml");
ClinicalDocument cda = CDAUtil.load(fis);

// Init an object of CCDTransformerImpl class, which implements the generic ICDATransformer interface.
// FHIR resource id generator can be either an incremental counter, or a UUID generator.
// The default is UUID; here it is set as COUNTER.
ICDATransformer ccdTransformer = new CCDTransformerImpl(IdGeneratorEnum.COUNTER);

// Finally, the CCD document instance is transformed to a FHIR Bundle, where the first entry is
// the Composition corresponding to the ClinicalDocument, and further entries are the ones referenced
// from the Composition.
Bundle bundle = ccdTransformer.transformDocument(cda);

// Through HAPI library, the Bundle can easily be printed in JSON or XML format.
FHIRUtil.printJSON(bundle, "src/test/resources/output/C-CDA_R2-1_CCD.json");
```

## Transforming a CDA document to a transactional bundle with Provenance 
```java
// Load MDHT CDA packages. Otherwise ContinuityOfCareDocument and similar documents will not be recognised.
// This has to be called before loading the document; otherwise will have no effect.
CDAUtil.loadPackages();

// Init an object of CCDTransformerImpl class, which implements the generic ICDATransformer interface.
// FHIR resource id generator can be either an incremental counter, or a UUID generator.
// The default is UUID; here it is set as COUNTER.
ICDATransformer ccdTransformer = new CCDTransformerImpl(IdGeneratorEnum.COUNTER);

// Create an identifier for the Provenance object identifying the running system.
Identifier id = new Identifier();
id.setValue("Data Processing Engine");

//Create an OpenHealthTools CCD document object to pass into the library by parsing an input file (or stream).
ContinuityOfCareDocument ccd = (ContinuityOfCareDocument) CDAUtil.loadAs(<inputStream>,
					ConsolPackage.eINSTANCE.getContinuityOfCareDocument());

//Load your input file into memory as a string (logic for this is beyond the scope of this example).
String rawDocument = <inputStream>

// The CCD document instance is transformed to a FHIR Bundle, which creates the Composition, Provenance, and documentReference objects.
Bundle bundle = ccdTransformer.transformDocument(cda, BundleType.TRANSACTION, null, rawDocument, identifier);

// Through HAPI library, the Bundle can easily be printed in JSON or XML format.
FHIRUtil.printJSON(bundle, "src/test/resources/output/C-CDA_R2-1_CCD.json");
```


## Transforming a CDA artifact (e.g. an entry class) to the corresponding FHIR resource(s)

```java
// Init an object of ResourceTransformerImpl class, which implements the IResourceTransformer
// interface. When instantiated separately from the CDATransformer context, FHIR resources are
// generated with UUID ids, and a default patient reference is added as "Patient/0"
IResourceTransformer resTransformer = new ResourceTransformerImpl();

// Assume we already have a CCD instance in the ccd object below (skipping CDA artifact creation from scratch)
// Traverse all the sections of the CCD instance
for(Section cdaSec: ccd.getSections()) {
    // Transform a CDA section to a FHIR Composition.Section backbone resource
    Composition.Section fhirSec = resTransformer.tSection2Section(cdaSec);

    // if a CDA section is instance of a Family History Section (as identified through its templateId)
    if(cdaSec instanceof FamilyHistorySection) {
        // cast the section to FamilyHistorySection
        FamilyHistorySection famSec = (FamilyHistorySection) cdaSec;
        // traverse the Family History Organizers within the Family History Section
        for(FamilyHistoryOrganizer fhOrganizer : famSec.getFamilyHistories()) {
            // Transform each C-CDA FamilyHistoryOrganizer instance to FHIR FamilyMemberHistory instance
            FamilyMemberHistory fmh = resTransformer.tFamilyHistoryOrganizer2FamilyMemberHistory(fhOrganizer);
        }
    }
}

// Again, any FHIR resource can be printed through FHIRUtil methods.
FHIRUtil.printXML(fmh, "src/test/resources/output/family-member-history.xml");
```

It should be noted that most of the time, IResourceTransformer methods return a FHIR Bundle composed of a few FHIR resources,
instead of a single FHIR resource as in the example above. For example, tProblemObservation2Condition method returns a Bundle
that contains the corresponding Condition as the first entry, which can also include other referenced resources such as Encounter, Practitioner.

Further examples can be found in [ResourceTransformerTest](./src/test/java/tr/com/srdc/cda2fhir/ResourceTransformerTest.java) class
and [CCDTransformerImpl](./src/main/java/tr/com/srdc/cda2fhir/transform/CCDTransformerImpl.java) class.

## Validating generated FHIR resources

We have also integrated the [HAPI FHIR Validator](http://hapifhir.io/doc_validation.html) and have implemented a wrapper interface and a class on top of this validator: IValidator and ValidatorImpl. A resource can be validated individually, or a Bundle
containing several resources as in the case of CDA transformation outcome can be validated at once. Validation outcome is provided as HTML within an OutputStream.

```java
// Init an object of ValidatorImpl class, which implements the IValidator interface.
IValidator validator = new ValidatorImpl();

// Assume we already have a Bundle object to be validated at hand. Call the validateBundle method
// of the validator and get the validation outcome as HTML in a ByteArrayOutputStream.
ByteArrayOutputStream valOutcomeOs = (ByteArrayOutputStream) validator.validateBundle(bundle);

// The HTML can be printed to a file.
FileOutputStream fos = new FileOutputStream(new File("src/test/resources/output/validation-result-w-profile-for-C-CDA_R2-1_CCD.html"));
valOutcomeOs.writeTo(fos);

// Close the streams
valOutcomeOs.close();
fos.close();
```

Further examples can be found in [ValidatorTest](./src/test/java/tr/com/srdc/cda2fhir/ValidatorTest.java) class.

## Acknowledgement
This research has received funding from the European Union’s Horizon 2020 research and innovation programme under grant agreement No 689181,
[C3-Cloud Project](http://www.c3-cloud.eu/) (A Federated Collaborative Care Cure Cloud Architecture for Addressing the Needs of Multi-morbidity and Managing Poly-pharmacy).

This research has received funding from the European Union’s Horizon 2020 research and innovation programme under grant agreement No 689444,
[POWER2DM Project](http://www.power2dm.eu/) (Predictive model-based decision support for diabetes patient empowerment).

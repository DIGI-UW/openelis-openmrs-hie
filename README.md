# OpenELIS Global - OpenMRS 3 HIE setup

A demo health information exchange: OpenMRS 3 orders lab tests, OpenELIS Global receives and results
them, and both exchange FHIR R4 resources through OpenHIM and a shared health record (SHR, HAPI FHIR).
It follows the OpenHIE [laboratory workflows](https://guides.ohie.org/arch-spec/introduction/laboratory-work-flows).

| Component | Version |
|---|---|
| OpenELIS Global | 3.3.0 (the [openelis-docker](https://github.com/DIGI-UW/openelis-docker) 3.3.0 stack, with the analyser Bridge 3.4.1) |
| OpenMRS | Reference Application 3.5.0, Lab on FHIR module 1.5.3 |
| OpenHIM | core 7.1.0, console 1.15.0 |
| SHR | HAPI FHIR 5.5.1 on PostgreSQL 14 |

Guide: [OpenMRS (Lab on FHIR)](https://uwdigi.atlassian.net/wiki/spaces/oeg/pages/1777467400)

## Start

```bash
chmod 1777 configs/openelis/analyzer-imports
docker compose up -d
```

The first start takes several minutes (OpenMRS builds its database). Give Docker enough memory for
OpenMRS, OpenELIS and two FHIR servers.

| Service | URL | User : password |
|---|---|---|
| OpenMRS 3 | <http://localhost/openmrs/spa> (also <https://localhost/openmrs/spa>) | `admin` : `Admin123` |
| OpenELIS Global | <https://localhost> | `admin` : `adminADMIN!` |
| OpenHIM console | <http://localhost:9000> (open <https://localhost:8085> once and accept its certificate) | `root@openhim.org` : `openhim` |
| SHR | <http://localhost:8090/fhir> | |

These host ports must be free: 80, 443, 2575, 3306, 5001, 5050-5052, 5432, 7788, 8080, 8081, 8085,
8088, 8090, 8282, 8443, 8444, 9000, 12001, 12011, 15432 and 27017. OpenELIS's
HTTP port moves to 8088 (`OE_HTTP_PORT` in `.env`) because the OpenMRS gateway uses 80.

## How the lab flow is wired

1. **Order.** A clinician orders a test in OpenMRS. Lab on FHIR sends a `Task` and `ServiceRequest`
   through OpenHIM (channel `central fhir server`, `/fhir`, client `OpenMRS`) to the SHR.
2. **Receive.** OpenELIS polls the SHR through the same channel every 20 seconds as client `OpenELIS`
   (`org.openelisglobal.remote.source.*` in `configs/openelis/properties/common.properties`), shows the
   order under **Order > Incoming Orders**, matches the tests on their LOINC codes and sets the remote
   `Task` to accepted.
3. **Result.** Enter and validate the results in OpenELIS. OpenMRS shows them on the order.

On a fresh install, Lab on FHIR applies its settings from the second start of OpenMRS: run
`docker compose restart backend` once OpenMRS is up, so patients reach OpenELIS with their ID as the
national ID. Before that, enter the national ID when you receive the order.

To test, register a new patient in OpenMRS (orders for the generated demo patients are skipped) and
order one of these tests, defined with their LOINC codes in
[configs/openelis/configuration/backend/tests/example-tests.csv](./configs/openelis/configuration/backend/tests/example-tests.csv):

| Test | LOINC |
|---|---|
| White Blood Cells Count (WBC) | 26464-8 |
| Red Blood Cells Count (RBC) | 23859-2 |
| Platelets | 26515-7 |
| Hemoglobin | 718-7 |

More: [EMR-LIS order communication](https://digi-uw.github.io/healthinformationexchange/lis-workflows/lis-workflows.html#tutorial-lab-order-communication-between-openmrs-and-openelis)
and the [demo video](https://www.youtube.com/watch?v=LsHhDrrlvKw).

## Configuration files

| File | Purpose |
|---|---|
| `configs/openelis/properties/common.properties` | OpenELIS: OpenHIM credentials, order polling, result push, provider import |
| `configs/openelis/properties/SystemConfiguration.properties` | `ACCEPT_EXTERNAL_ORDERS=true` |
| `configs/openelis/configuration/backend/` | Test catalogue loaded at startup |
| `configs/openhim/configs/openhim-config.json` | OpenHIM clients `OpenELIS` and `OpenMRS` and the two channels |
| `configs/openmrs/distro/configuration/globalproperties/openelis-openmrs.xml` | Lab on FHIR settings |

Everything else about the OpenELIS part (certificates, ports, backups, the analyser Bridge) works as in
openelis-docker: see its [README](https://github.com/DIGI-UW/openelis-docker#readme).

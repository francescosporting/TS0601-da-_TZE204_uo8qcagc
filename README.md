"""Tuya CO/Gas sensor."""
from typing import Dict

from zigpy.profiles import zha
from zigpy.quirks import CustomDevice
import zigpy.types as t
from zigpy.zcl.clusters.general import Basic, Ota, Time, Groups, Scenes
from zigpy.zcl.clusters.measurement import CarbonMonoxideConcentration, CarbonDioxideConcentration
from zigpy.zcl.clusters.security import IasZone

from zhaquirks.const import (
    DEVICE_TYPE,
    ENDPOINTS,
    INPUT_CLUSTERS,
    MODELS_INFO,
    OUTPUT_CLUSTERS,
    PROFILE_ID,
)
from zhaquirks.tuya import TuyaLocalCluster, TuyaManufCluster
from zhaquirks.tuya.mcu import DPToAttributeMapping, TuyaMCUCluster

ZONE_TYPE = 0x0001

class TuyaCarbonMonoxideConcentration(CarbonMonoxideConcentration, TuyaLocalCluster):
    """Tuya local CarbonMonoxideConcentration cluster."""

class TuyaCarbonMonoxideDetectorZone(IasZone, TuyaLocalCluster):
    """IAS Zone."""
    _CONSTANT_ATTRIBUTES = {ZONE_TYPE: IasZone.ZoneType.Carbon_Monoxide_Sensor}

class TuyaMethaneConcentration(CarbonDioxideConcentration, TuyaLocalCluster):
    """Tuya local MethaneConcentration cluster."""

class TuyaMethaneDetectorZone(IasZone, TuyaLocalCluster):
    """IAS Zone."""   
    _CONSTANT_ATTRIBUTES = {ZONE_TYPE: IasZone.ZoneType.Fire_Sensor}

class CoMethaneManufCluster(TuyaMCUCluster):

    attributes = TuyaMCUCluster.attributes.copy()

    dp_to_attribute: Dict[int, DPToAttributeMapping] = {
        1: DPToAttributeMapping(
            TuyaMethaneDetectorZone.ep_attribute,
            "zone_status",
            lambda x: IasZone.ZoneStatus.Alarm_1 if not x else 0,
            endpoint_id=2
        ),
        2: DPToAttributeMapping(
            TuyaMethaneConcentration.ep_attribute,
            "measured_value",
            lambda x: x * 1e-5
        ),
        18: DPToAttributeMapping(
            TuyaCarbonMonoxideDetectorZone.ep_attribute,
            "zone_status",
            lambda x: IasZone.ZoneStatus.Alarm_1 if not x else 0
        ),
        19: DPToAttributeMapping(
            TuyaCarbonMonoxideConcentration.ep_attribute,
            "measured_value",
            lambda x: x * 1e-8
        )
    }

    data_point_handlers = {
        1: "_dp_2_attr_update",
        2: "_dp_2_attr_update",
        18: "_dp_2_attr_update",
        19: "_dp_2_attr_update"
    }

class CoMethaneSensor(CustomDevice):
    """Tuya CO/Gas sensor."""

    signature = {
        MODELS_INFO: [("_TZE204_uo8qcagc", "TS0601")],
        ENDPOINTS: {
            # endpoints=1 profile=260 device_type=0x0051
            # in_clusters=[0x0000, 0x0004, 0x0005, 0xef00],
            # out_clusters=[0x000a, 0x0019]
            1: {
                PROFILE_ID: zha.PROFILE_ID,
                DEVICE_TYPE: zha.DeviceType.SMART_PLUG,
                INPUT_CLUSTERS: [
                    Basic.cluster_id,
                    Groups.cluster_id,
                    Scenes.cluster_id,
                    TuyaManufCluster.cluster_id,
                ],
                OUTPUT_CLUSTERS: [
                    Time.cluster_id,
                    Ota.cluster_id,
                ],
            },
            242: {
                PROFILE_ID: 0xa1e0,
                DEVICE_TYPE: 0x0061,
                INPUT_CLUSTERS: [],
                OUTPUT_CLUSTERS: [
                    0x0021
                ]
            }
        },
    }

    replacement = {
        ENDPOINTS: {
            1: {
                PROFILE_ID: zha.PROFILE_ID,
                DEVICE_TYPE: zha.DeviceType.ON_OFF_LIGHT,
                INPUT_CLUSTERS: [
                    Basic.cluster_id,
                    Groups.cluster_id,
                    Scenes.cluster_id,
                    CoMethaneManufCluster,
                    TuyaCarbonMonoxideConcentration,
                    TuyaCarbonMonoxideDetectorZone,
                    TuyaMethaneConcentration,
                ],
                OUTPUT_CLUSTERS: [
                    Time.cluster_id,
                    Ota.cluster_id,
                ],
            },
            2: {
                PROFILE_ID: zha.PROFILE_ID,
                DEVICE_TYPE: zha.DeviceType.IAS_ZONE,
                INPUT_CLUSTERS: [
                    TuyaMethaneDetectorZone
                ],
                OUTPUT_CLUSTERS: [],
            },
            242: {
                DEVICE_TYPE: 0x0061,
                INPUT_CLUSTERS: [],
                OUTPUT_CLUSTERS: [
                    0x0021
                ]
            }
        }
    }

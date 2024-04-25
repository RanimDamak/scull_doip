# scull_doip

use kernel::{
    file::{flags, File, Operations},
    io_buffer::{IoBufferReader, IoBufferWriter},
    miscdev,
    prelude::*,
    sync::{smutex::Mutex, Arc, ArcBorrow},
    Module,
};


pub fn to_bytes32(input: &[u32]) -> Vec<u8> {

    let mut output = Vec::<u8>::new();
    input.iter().for_each(|val| { let _ = output.try_extend_from_slice(&val.to_be_bytes()); });

    output
}
pub fn to_bytes16(input: &[u16]) -> Vec<u8> {

    let mut output = Vec::<u8>::new();
    input.iter().for_each(|val| { let _ = output.try_extend_from_slice(&val.to_be_bytes()); });

    output
}
module! {
    type: VDev,
    name: "vdev",
    license: "GPL",
    params: {
        devices: u32 {
            default: 1,
            permissions: 0o644,
            description: "Number of virtual devices",
        },
    },
}
struct Device {
    number: usize,
    contents: Mutex<Vec<u8>>,
}

struct VDev {
    _devs: Vec<Pin<Box<miscdev::Registration<VDev>>>>,
}
struct Doipframe  {
    protocol_version: u8,
    invprotocol_verison: u8,
    payload_type: Vec<u16>,
    payload_length: Vec<u32>,
    payload_data:Vec<u8>,
}
#[vtable]
impl Operations for VDev {
    type OpenData = Arc<Device>;
    type Data = Arc<Device>;

    fn open(context: &Arc<Device>, file: &File) -> Result<Arc<Device>> {
        pr_info!("File for device {} was opened\n", context.number);
        if file.flags() & flags::O_ACCMODE == flags::O_WRONLY {
            context.contents.lock().clear();
        }
        Ok(context.clone())
    }

    fn read(
        data: ArcBorrow<'_, Device>,
        _file: &File,
        writer: &mut impl IoBufferWriter,
        offset: u64,
    ) -> Result<usize> {
        pr_info!("File for device {} was read\n", data.number);
        let offset = offset.try_into()?;
        let vec = data.contents.lock();

        let len = core::cmp::min(writer.len(), vec.len().saturating_sub(offset));
        pr_info!("-------------------\n");
        writer.write_slice(&vec[offset..][..len])?;
        Ok(len)
    }

    fn write(
        data: ArcBorrow<'_, Device>,
        _file: &File,
        reader: &mut impl IoBufferReader,
        offset: u64,
    ) -> Result<usize> {
        pr_info!("File for device {} was written\n", data.number);
        let offset = offset.try_into()?;
        let len = reader.len();
        let new_len = len.checked_add(offset).ok_or(EINVAL)?;
        let mut vec = data.contents.lock();
        if new_len > vec.len() {
            vec.try_resize(new_len, 0)?;
        }

        let mut frame = Doipframe{
            protocol_version: 2,// prtocol version 0x02hex
            invprotocol_verison: 2^0xFF,//protocol inversion 0x02^0XFF
            payload_type: Vec::new(),//DoIP payload type Diagnostic message
            payload_length: Vec::new(),//DoIP payload type Diagnostic message
            payload_data:Vec::from(reader.read_all()?),//DoIP payload length
        };
        let _ = frame.payload_type.try_push(32769); //0x8001
        
        let x = frame.payload_data.len().try_into().unwrap();
        let _ = frame.payload_length.try_push(x) ;

        let v = to_bytes16(&frame.payload_type[0..frame.payload_type.len()]);
        vec.try_push(frame.protocol_version)?; 
        vec.try_push(frame.invprotocol_verison)?; 
        vec.try_push(v[0])?; 
        vec.try_push(v[1])?;

        let v = to_bytes32(&frame.payload_length[0..frame.payload_length.len()]);
        vec.try_push(v[0])?;
        vec.try_push(v[1])?;
        vec.try_push(v[2])?;
        vec.try_push(v[3])?;

        for elt in &frame.payload_data {
            vec.try_push(*elt)?;
            pr_info!("element: {}\n", *elt);
        }

        Ok(len)
    }
}

impl Module for VDev {
    fn init(_name: &'static CStr, module: &'static ThisModule) -> Result<Self> {
        let count = {
            let lock = module.kernel_param_lock();
            (*devices.read(&lock)).try_into()?
        };
        pr_info!("-----------------------\n");
        pr_info!("starting {} vdevices!\n", count);
        pr_info!("watching for changes...\n");
        pr_info!("-----------------------\n");
        let mut devs = Vec::try_with_capacity(count)?;
        for i in 0..count {
            let dev = Arc::try_new(Device {
                number: i,
                contents: Mutex::new(Vec::new()),
            })?;
            let reg = miscdev::Registration::new_pinned(fmt!("vdev{i}"), dev)?;
            devs.try_push(reg)?;
        }
        Ok(Self { _devs: devs })
    }
}
